# Getting Started with CTSM

This guide walks from zero to a CTSM sandbox that has all submodules in place
and a working Python environment. Once finished, see
`reference/running-cases.md` for the CIME case workflow that turns the sandbox
into a running simulation.

---

## Step 1: Understand the repository

CTSM (the **Community Terrestrial Systems Model**) is the land component of
CESM and includes the **Community Land Model (CLM)**. The repository
`ESCOMP/CTSM` is the canonical source.

| Item | Value |
|------|-------|
| URL | https://github.com/ESCOMP/CTSM |
| Default branch | `master` |
| Latest science tag at time of writing | `ctsm5.4.002` (CLM6 physics default) |
| License | CESM Copyright (BSD-style), see top-level `Copyright` file |
| Documentation site | https://escomp.github.io/CTSM/ |
| Wiki | https://github.com/ESCOMP/CTSM/wiki |
| Discussion forum | https://bb.cgd.ucar.edu/cesm/forums/ctsm-clm-mosart-rtm.134/ |

CTSM is bundled with a copy of CIME (Common Infrastructure for Modeling Earth)
and the active CESM components needed to drive land-only simulations:
CMEPS (NUOPC mediator), CDEPS (data models including DATM), CISM
(land ice), MOSART and mizuRoute (river routing), RTM (legacy river), and
FATES (ecosystem demography). Each lives at a pinned tag in `.gitmodules`.

---

## Step 2: Clone the repository

```bash
git clone https://github.com/ESCOMP/CTSM.git my_ctsm_sandbox
cd my_ctsm_sandbox
```

This is a plain `git clone`. It does **not** pull submodules. You will see
empty directories at `cime/`, `components/cmeps/`, `src/fates/`, etc. until
the next step.

If you want a specific tag:

```bash
git clone -b ctsm5.4.002 https://github.com/ESCOMP/CTSM.git my_ctsm_sandbox
```

---

## Step 3: Run `git-fleximod update`

CTSM uses **git-fleximod** (a wrapper around git submodule, maintained at
ESMCI/git-fleximod) instead of plain submodules so it can manage tag pins.
The script lives in `bin/git-fleximod`.

```bash
./bin/git-fleximod update
./bin/git-fleximod --help    # full options
```

This populates:

| Path | What it is | Pinned via |
|------|-----------|------------|
| `cime/` | CIME case scripts and infrastructure | `fxtag = cime6.1.146` |
| `ccs_config/` | CIME grids, machines, compsets for CESM | `fxtag = ccs_config_cesm1.0.77` |
| `components/cmeps/` | NUOPC-based CESM mediator | versioned in `.gitmodules` |
| `components/cdeps/` | CESM data models (DATM, DICE, DOCN, DROF) | versioned in `.gitmodules` |
| `components/cism/` | CISM-wrapper ice sheet | `fxtag = cismwrap_2_2_013` |
| `components/mosart/` | River routing | `fxtag = mosart1.1.13` |
| `components/mizuroute/` | Reach-based river routing | `fxtag = v3.0.1` |
| `components/rtm/` | Legacy River Transport Model | `fxtag = rtm1_0_89` |
| `src/fates/` | FATES ecosystem demography | `fxtag = sci.1.89.0_api.43.0.0` |

You **must rerun** `./bin/git-fleximod update` whenever `.gitmodules` changes,
which typically happens after `git checkout` of a different tag or branch, or
after `git merge` brings in upstream changes that bump a submodule pin.

To check whether your sandbox is in sync:

```bash
./bin/git-fleximod status
```

A leading `s` in the status column means a submodule is out of sync with its
pinned tag.

---

## Step 4: Pick a machine

CIME ships with port files for many HPC systems under
`ccs_config/cesm/machines/config_machines.xml`. To list them:

```bash
cd cime/scripts
./query_config --machines
```

| Machine | Notes |
|---------|-------|
| `derecho` | NCAR/CISL primary supercomputer, fully tested, all CTSM tools work |
| `casper` | NCAR analysis cluster, good for `subset_data` and post-processing |
| `izumi` | NCAR small cluster, regression testing baseline machine |
| `perlmutter` | NERSC, GPU partition supported via `--queue` |
| `frontera`, `lonestar6` | TACC |
| `pleiades` | NASA HEC |
| Many others | Run `query_config --machines` for the current list |

For an unsupported machine you must port CIME, see
https://github.com/ESMCI/cime/wiki/Porting-Overview.

---

## Step 5: Create the Python environment (`ctsm_pylib`)

Several CTSM tools (`subset_data`, `mesh_maker`, `fsurdat_modifier`,
`mksurfdata_esmf` namelist generation, `run_sys_tests`, the doc build, the
PR helper scripts) need a Python environment with specific package versions.
A helper script creates this for you:

```bash
./py_env_create
conda activate ctsm_pylib
```

This installs the environment described by `python/conda_env_ctsm_py.yml`
(currently Python 3.13.2). On Derecho and Casper you may need to first run
`module load conda` to put `conda` on your `PATH`.

If you already have a `ctsm_pylib` environment from before `ctsm5.3.040`, the
`py_env_create` script will detect the version mismatch. To rename your old
environment instead of overwriting:

```bash
./py_env_create -r ctsm_pylib_old
```

This renames the existing environment to `ctsm_pylib_old` and installs the
new one as `ctsm_pylib`. See `python/README.updating_ctsm_pylib.md`.

---

## Step 6: Verify the sandbox

```bash
ls cime/scripts/create_newcase                # should exist and be executable
ls src/main/clm_driver.F90                    # should exist
ls src/fates/main/EDMainMod.F90               # should exist if FATES is in
./bin/git-fleximod status                     # all rows clean (no leading s)
```

If `src/fates/` looks empty, FATES did not come in. Rerun:

```bash
./bin/git-fleximod update fates
```

---

## Step 7: Sanity-build a tiny case (optional)

This is the smallest end-to-end exercise to confirm everything is wired
together. The full case workflow is in `reference/running-cases.md`.

```bash
cd cime/scripts
./create_newcase --case ~/cases/sanity_iclm60sp \
                 --res f10_f10_mg37 \
                 --compset I2000Clm60SpRs \
                 --mach <your-machine>
cd ~/cases/sanity_iclm60sp
./case.setup
./case.build         # 5 to 30 minutes depending on machine
./xmlchange STOP_OPTION=ndays,STOP_N=1
./case.submit
```

A 1-day `f10_f10_mg37` (10 degree x 10 degree) case finishes in a couple of
minutes on a login node and produces history files under `$RUNDIR`. Check
with:

```bash
./xmlquery RUNDIR
ls $(./xmlquery -value RUNDIR)/*.clm2.h0*.nc
```

---

## Common install pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cime/scripts/create_newcase` not found | Skipped `./bin/git-fleximod update` | Run it now |
| `src/fates/` empty | Same | `./bin/git-fleximod update fates` |
| `git-fleximod status` shows `s` rows | You changed branches without re-updating | `./bin/git-fleximod update` |
| `modulefile not found` for your machine | Machine has not been ported to CIME | Port CIME or pick a supported machine |
| `conda: command not found` | Conda not on `PATH` | `module load conda` (Derecho/Casper) or install Miniforge |
| `py_env_create` fails to solve | Old `ctsm_pylib` blocks it | Use `./py_env_create -r ctsm_pylib_old` |
| `gen_mksurfdata_build` works only on Derecho | Known limitation, see issue #2341 | Use `subset_data` on other machines |

---

## Where to next

- Run a real CTSM case: `running-cases.md`
- Understand what is inside `src/`: `architecture.md`
- Pick the right physics version and options: `physics-and-options.md`
- Make a custom surface dataset for a site: `python-tools.md`
