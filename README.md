# CTSM Skill

A progressive-disclosure skill for the
[Community Terrestrial Systems Model (CTSM)](https://github.com/ESCOMP/CTSM),
the land component of CESM that includes the Community Land Model (CLM).

> **Maintained by:** the NCAR/CGD CTSM software engineering and science teams (https://github.com/ESCOMP/CTSM)
> **Skill author:** Koutian Wu (ktwu01@gmail.com)
> **Skill version:** 0.1.0

> ⚠️ **Disclaimer — please read before using this skill.**
> This skill is **not a gold-standard reference**. It is a helper that lowers
> the barrier for new users to **get their hands dirty** with the model. AI
> agents (and the humans drafting this material) make mistakes; commands, file
> paths, namelist options, and physics explanations here can be wrong,
> incomplete, or out of date. **Always cross-check with the official model
> documentation, the source code, and a human expert before trusting any
> output for research, publication, or operational use.**

## What This Is

A self-contained knowledge package that teaches AI agents (and humans) how to
**clone, build, run, modify, debug, and contribute to** CTSM, covering the
ctsm5.4 series with CLM6 physics defaults, the CIME case workflow, the
FATES ecosystem demography submodule, LILAC standalone coupling, the
`tools/` pre-processors, and the ESCOMP fork-and-PR workflow.

The skill captures the **procedural knowledge** that is normally only
transmitted by working alongside an experienced CTSM developer: when to rerun
`./bin/git-fleximod update`, how compset aliases like `I2000Clm60BgcCrop`
decompose into time period plus active and stub components, why history files
now split into `hXa` and `hXi`, what `LND_TUNING_MODE` actually controls, and
the order of `ChangeLog`, `ChangeSum`, and `ExpectedTestFails.xml` updates
that an integrator expects to see in a PR.

**Progressive disclosure:**
- `SKILL.md`, the routing hub: decision tree, repo layout, quick start, critical rules
- `reference/*.md`, deep-dive docs loaded on demand

## Contents

| Document | What's inside |
|----------|---------------|
| `SKILL.md` | Entry point: decision tree, repo layout, quick start, critical rules |
| `reference/getting-started.md` | Clone, `git-fleximod update`, supported machines, conda env |
| `reference/architecture.md` | `src/` module families, subgrid hierarchy, FATES, drivers |
| `reference/running-cases.md` | `create_newcase`, compsets, `case.setup`, build, submit |
| `reference/physics-and-options.md` | CLM6 vs CLM5 vs CLM4.5, BGC, SP, FATES, MOSART, urban, lake, fire |
| `reference/output-history-fields.md` | h0a/h0i tapes, `hist_fincl`, frequency, vector output |
| `reference/lilac-coupling.md` | LILAC purpose, ESMF API, atm_driver test |
| `reference/python-tools.md` | mksurfdata_esmf, subset_data, fsurdat_modifier, mesh_maker |
| `reference/debugging.md` | Build, runtime, restart, baselines |
| `reference/contributing-pr.md` | Fork, branch, PR, ChangeLog, ExpectedTestFails |

## Sources

This skill is grounded in:

1. **ESCOMP/CTSM** repository (master branch, ctsm5.4.x), README, README_GITFLEXIMOD.rst, README.CHECKLIST.new_case, WhatsNewInCTSM5.4.md, CONTRIBUTING.md
2. **`doc/`** in the CTSM repository: Quickstart.GUIDE.md, IMPORTANT_NOTES.md, README.NUOPC_driver.md, ChangeLog/ChangeSum, README.CHECKLIST.master_tags.md
3. **`bld/namelist_files/namelist_definition_ctsm.xml`** for namelist semantics
4. **`tools/`** READMEs: mksurfdata_esmf, modify_input_files, site_and_regional
5. **`lilac/docs/`** Sphinx source
6. **CTSM Sphinx docs**: https://escomp.github.io/CTSM/
7. **CTSM wiki**: https://github.com/ESCOMP/CTSM/wiki
8. **Lawrence et al. 2019**, "The Community Land Model Version 5", JAMES, doi:10.1029/2018MS001583

## Install

This skill follows the same layout as
[laps-skill](https://github.com/huangzesen/laps-skill) and the noahmp-skill
sibling:

```
ctsm-skill/
├── SKILL.md         ← routing hub (read first)
├── README.md        ← this file
├── LICENSE          ← MIT
├── .gitignore
└── reference/       ← deep-dive docs
```

To use with a Claude Code or LingTai agent, drop the directory into your
skills library and refresh.

## License

MIT for the skill content (this repository). CTSM itself is governed by the
CESM Copyright file in the ESCOMP/CTSM repository, see
https://github.com/ESCOMP/CTSM/blob/master/Copyright.
