# Contributing to ESCOMP/CTSM

CTSM uses a fork-and-pull-request workflow on GitHub, with a small number
of integrators on the software engineering team who do the final merge to
master and tag the result. This document covers the contributor side: how
to open an issue, set up a fork, prepare a branch, run tests, write a
ChangeLog entry, and submit a PR that an integrator can land without
rework.

Authoritative references in the repo:

- `CONTRIBUTING.md`, the short version of this document.
- `doc/README.CHECKLIST.master_tags.md`, the integrator checklist.
- `https://github.com/ESCOMP/CTSM/wiki`, the developer wiki.
- `https://github.com/ESCOMP/CTSM/wiki/Quick-start-to-CTSM-development-with-git`
- `https://github.com/ESCOMP/CTSM/wiki/Recommended-git-setup`
- `https://github.com/ESCOMP/CTSM/wiki/CTSM-coding-guidelines`
- `https://github.com/ESCOMP/CTSM/wiki/CTSM-development-workflow`
- The CTSM Developers Guide on the NCAR wiki:
  https://wiki.ucar.edu/display/ccsm/Community+Land+Model+Developers+Guide

---

## Step 0: Open an issue first

For anything beyond a tiny typo fix, open an issue at
https://github.com/ESCOMP/ctsm/issues before writing code. The issue is
where the design discussion happens. CTSM is a slow-moving research code
with a small core team; surprise PRs that overlap with planned work are
likely to be deferred or rejected.

If you cannot describe your change in an issue (e.g., a sensitive
algorithmic change), email `ctsm-software@ucar.edu` and ask to coordinate.

Subscribe to the ctsm-dev Google group for tag and development
notifications: https://groups.google.com/a/ucar.edu/forum/#!forum/ctsm-dev.

---

## Step 1: Fork and set up remotes

```bash
# On https://github.com/ESCOMP/CTSM, click "Fork" to your account, then:

git clone https://github.com/<you>/CTSM.git my_ctsm
cd my_ctsm
git remote add escomp https://github.com/ESCOMP/CTSM.git
git remote -v
# origin   https://github.com/<you>/CTSM.git (fetch / push)
# escomp   https://github.com/ESCOMP/CTSM.git (fetch / push)

./bin/git-fleximod update
```

Adopt the recommended git settings from
https://github.com/ESCOMP/CTSM/wiki/Recommended-git-setup. Most importantly,
configure `git pull` to default to fast-forward only on `master`, so you
cannot accidentally commit to a local `master` branch.

---

## Step 2: Make a feature branch off master

```bash
git checkout master
git pull --ff-only escomp master
git checkout -b add-foo-physics
```

Naming convention is loose. Keep it descriptive (`fix-snowfall-units`,
`add-li2024-tuning`, `subset-data-batch-mode`).

---

## Step 3: Make the change

Coding guidelines:
https://github.com/ESCOMP/CTSM/wiki/CTSM-coding-guidelines.
Highlights:

- Free-form Fortran 2003+ syntax.
- Module-per-file with the file named `<ModuleName>Mod.F90` (this is the
  near-universal CTSM convention).
- Use the existing derived-type containers; do not introduce new global
  variables.
- Write history-output `hist_addfld*` calls in the `Init` routine of the
  type that owns the field.
- Add `restart` machinery if the field needs to persist across restarts.
- Run any new physics through the conservation checks in
  `src/biogeophys/BalanceCheckMod.F90` and `src/biogeochem/CNBalanceCheckMod.F90`.
- For new namelist variables, add an entry to
  `bld/namelist_files/namelist_definition_ctsm.xml` (definition, type,
  category) and a default in `bld/namelist_files/namelist_defaults_ctsm.xml`.

For a new FATES feature, the CTSM-side change is usually small (a flag in
`user_nl_clm` and a switch in `clmfates_interfaceMod.F90`). The bulk of
the change goes to NGEET/fates as a separate PR. Then update the FATES
`fxtag` in `.gitmodules` once both PRs are ready, and submit the CTSM PR.

---

## Step 4: Run tests

The standard test sweep is `run_sys_tests` (at the top of CTSM). It submits
the CTSM regression test suite (build + short run + answer comparison
against a baseline) on Derecho and/or Izumi:

```bash
./run_sys_tests --help
```

If you do not have access to Derecho or Izumi, run a small subset locally:

```bash
cd cime/scripts
./create_test SMS.f10_f10_mg37.I2000Clm60Sp.<machine>_<compiler>
```

For unit tests:

```bash
cd src
make test                       # uses pFUnit, see src/README.unit_testing
```

Inspect `../run_sys_test.log`. Make sure:

- All previously passing tests still pass.
- Any new failures are either fixed or registered in
  `cime_config/testdefs/ExpectedTestFails.xml`.
- Any baseline changes are explained in the ChangeLog (next step).
- `./bin/git-fleximod status` is clean.

If your change touches answers, you must also generate or compare against
the relevant baselines. The integrator team will regenerate baselines on
official testing machines once the PR is merged; you should however
compare against existing baselines and document the diffs.

---

## Step 5: Update ChangeLog and ChangeSum

Every CTSM tag has a numbered entry in `doc/ChangeLog` and a one-line
entry in `doc/ChangeSum`. Contributors append a draft entry to `ChangeLog`;
the integrator finalizes during the tag step.

Use the helper:

```bash
cd doc
./UpdateChangelog.pl <TAGNAME> "<one-line summary>"
```

This opens `ChangeLog` in your editor with a new section pre-filled.
Fields to fill (if relevant to your PR):

- **Purpose of changes**
- **Bugs fixed** (issue numbers)
- **Notes of particular relevance for users**
- **Notes of particular relevance for developers**
- **Testing summary** (which test suites passed, which failed and why)
- **Answer changes** (bit-for-bit, or which compsets/grids change answers)
- **Issues fixed** (list with `#NNN`)
- **Pull Requests that document the changes** (`#NNNN`)

Then update the date stamp:

```bash
./UpdateChangelog.pl -update
```

Commit `doc/ChangeLog` and `doc/ChangeSum` along with your code change.

If the parameter file changed, include the output of `compare_paramfiles`
in the ChangeLog entry.

---

## Step 6: Push and open the PR

```bash
git push origin add-foo-physics
```

On GitHub, open a PR from `<you>:add-foo-physics` to `ESCOMP:master`.
Click **compare across forks** if not auto-detected. Fill in the PR
template:

- **Description**: one paragraph summary plus link to the issue.
- **Type of change**: bug fix, enhancement, documentation, etc.
- **Code review checklist**: tests run, ChangeLog updated, no spurious
  formatting changes.

Add reviewers from the CTSM software team (see top-level `README.md`):

- Erik Kluzek (`@ekluzek`)
- Bill Sacks (`@billsacks`)
- Sam Levis (`@slevis-lmwg`)
- Adrianna Foster (`@adrifoster`)
- Sam Rabin (`@samsrabin`)
- Greg Lemieux (`@glemieux`)
- Ryan Knox (`@rgknox`)

The science team (Will Wieder, Dave Lawrence, Danica Lombardozzi, Keith
Oleson, Sean Swenson, Peter Lawrence, Gordon Bonan) reviews science
content. For FATES PRs, also tag the FATES leads.

In most cases you will not merge the PR yourself. An integrator will run
their own additional testing and merge.

---

## Step 7: Iterate on review

CTSM reviewers are thorough. Common feedback themes:

- **Restart support**: any new persistent state must be added to the
  restart file via `restartvar` calls.
- **Subgrid level correctness**: do not loop over patches when you mean
  columns, or vice versa.
- **Initialization order**: derived types must be initialized before any
  module reads from them.
- **Style**: 100-character lines, two-space indent in new code, lowercase
  Fortran intrinsics, no `implicit none` at the subroutine level if the
  module already has `implicit none`.
- **Unit tests**: every new module is expected to ship at least one pFUnit
  test under `src/<dir>/test/`.

Push commits to your branch; the PR updates automatically. Avoid force
pushes after review has started; reviewers will lose the diff context.

---

## Step 8: After merge (integrator side)

This is what an integrator will do, copied from
`doc/README.CHECKLIST.master_tags.md`:

1. Update sandbox to latest `master` and verify `git-fleximod status`.
2. Run full test sweep on their fork's feature branch.
3. Update `ExpectedTestFails.xml` for any new expected failures.
4. Update `ChangeLog` (finalize) and date stamp.
5. Commit, push, open PR.
6. Merge PR.
7. Verify `master` matches the merged branch (`git diff` shows nothing).
8. Make annotated tag: `git tag -a <TAGNAME>`.
9. Push tag: `git push escomp <TAGNAME>`.
10. Update the upcoming-tags project board.
11. Add the tag to the CESM test database.

You do not run these as a contributor. They are listed here so you know
what to expect after your PR is merged.

---

## Special cases

### FATES PRs

Submit the FATES change to https://github.com/NGEET/fates first. Once that
is merged and tagged, submit a CTSM PR that:

- Bumps the `fxtag` for FATES in `.gitmodules`.
- Adds any CTSM-side namelist or interface changes.
- Updates the parameter file if API changed.

The HLM-FATES compatibility table at
https://fates-users-guide.readthedocs.io/en/latest/user/release-tags-compat-table.html
must reflect the new pairing.

### Tools-only PRs

Changes restricted to `tools/` or `python/ctsm/` do not require a full
`run_sys_tests` sweep, but you should still run `python/run_ctsm_py_tests`
and confirm that any tool with a system test (e.g., subset_data) still
passes:

```bash
cd python
make all
```

(`make all` runs unit tests, system tests, and pylint. You need
`module load nco` first.)

### Documentation-only PRs

Edit Sphinx sources under `doc/source/`. Build locally with `cd doc &&
./build_docs --help` to confirm rendering. Documentation PRs do not need
ChangeLog entries.

---

## Where to next

- Source layout when looking for the right place to put a new module: `architecture.md`
- Pre-PR sanity testing: `debugging.md`
- Find an existing namelist option to model your new one on: `physics-and-options.md`
- Write history-output for your new variable correctly: `output-history-fields.md`
