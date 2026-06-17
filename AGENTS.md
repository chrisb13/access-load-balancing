# Prompt to send to AI tool

You are helping perform a preliminary performance profiling and optimisation pass for an ACCESS-OM3 / MOM6-CICE6 / MOM6-CICE6-WW3 / standalone-component configuration on NCI/Gadi.

This is a **single-prompt, multi-gate workflow**. Do not wait for me to provide a new prompt after every stage. Continue through the stages below, but stop at each approval gate before changing model configuration in a risky way or submitting jobs.

The goal is not blind tuning. The goal is to use ESMF profile summaries, Payu/PBS logs, component logs, and, where useful, `access-profiling` / `esmf-trace` to identify the first safe optimisation family for this config.

Very important: this workflow must produce and keep updating both:

1. A **progress Markdown file**, updated after every meaningful step and **after every completed run**.
2. A **re-runnable performance summary notebook**, with plots, tables, and a GitHub-issue-ready optimisation story.

Do not leave plotting and documentation until the end. Start the structure early, then keep it updated after every meaningful Codex response, every approved/rejected idea, and every completed run.

---

## Case input block

Use these values for this optimisation thread:

```text
CONFIG_NAME = "<replace with config name, e.g. dev-MC_25km_jra_iaf or MCW_100km_era5_iaf_KPP>"

GITHUB_BRANCH_URL = "<replace with branch URL, e.g. https://github.com/ACCESS-NRI/access-om3-configs/tree/dev-MC_25km_jra_iaf>"

LOCAL_CONFIG_PATH = "<replace with local Gadi path to the source/reference config (e.g. a git checkout of the ACCESS-OM3 config repo)>"

PROJECT_PATH = "<replace with top-level project directory; the baseline run, all test runs, and profiling_analysis/ all reside directly inside here>"

ESMF_TRACE_PATH = "/g/data/tm70/ek4684/esmf-trace"

EXPECTED_RUN_COMMAND = "inspect first; usually: module use /g/data/vk83/modules && module load payu && payu run"

OPTIMISATION_OBJECTIVE = "preliminary optimisation using ESMF profile summary evidence; optimise both wall-clock and CPU-hours/model-year, but prioritise safe cost reduction if wall-clock is not harmed much"

MAX_NEW_TEST_RUNS_FOR_PRELIMINARY_PASS = 3

PROGRESS_MD = "<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_optimisation_progress.md"

NOTEBOOK_PATH = "<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_summary.ipynb"

GITHUB_ISSUE_DRAFT = "<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_github_issue_draft.md"

# MOM6 finer-resolution mask table settings. Leave as auto-detect unless you know the answer.
FINE_RESOLUTION_MOM_MASKTABLE_REQUIRED = "auto-detect; true for 25km and 8km MOM6/OCN configs; usually false for coarser configs"

OM3_SCRIPTS_PATH = "<replace with local om3-scripts repo path if known; otherwise locate or clone/access ACCESS-NRI/om3-scripts locally>"

MOM_MASKTABLE_SCRIPT = "masktable_generation/gen_masktable.sh"

MOM_MASKTABLE_POLICY = "If OCN/MOM PE count or LAYOUT changes for 25km or 8km configs, generate a matching mask table, place/copy it into the experiment configuration directory where possible, and update MOM_input plus config.yaml consistently."
```

---

## Hard constraints

1. Do not change science settings unless explicitly asked.
2. Do not change output frequency.
3. Do not change restart frequency.
4. Do not change output fields.
5. Do not remove active components.
6. Do not change forcing paths unless required to fix a broken run, and ask first.
7. Do not change run length unless it is part of an explicitly proposed and approved test.
8. Do not delete existing outputs, restarts, archives, work directories, or logs.
9. Do not submit jobs without explicit approval.
10. Keep all edits reviewable with `git diff`.
11. Treat this as a coupled/load-balancing problem unless the config inspection proves it is standalone.
12. Every accepted change must be traceable in the progress Markdown and run registry.
13. Every completed run must be added to the performance summary CSV/JSON files and the notebook.
14. For finer-resolution MOM6/OCN configs, especially 25 km and 8 km, do **not** change `ocn_ntasks`, the MOM `LAYOUT`, or the OCN PET layout without checking whether a new MOM mask table is required.
15. If a new MOM mask table is required, update `MOM_input` and `config.yaml` consistently and keep the generated mask table in the experiment/configuration directory where possible.
16. This workflow has been tested on Claude, learn from past mistakes/corrections by reading all these files and following the advice within: `.claude/memory/MEMORY.md`. Also add to these memory files when new mistakes and corrections are needed.
17. Minimise the number of times you need to ask for approval when interacting with Gadi or the local shell. Where possible combine up Gadi/shell queries into one large prompt so as to minimise human-needed approval.
18. When changing the total number of nodes, try to keep the ratio of ocn to non_ocn cores similar to the baseline.
19. As we progress through the 11 stages, announce to the user which stage we are in.

---

## Creating OM3 experiments

The following details how to create experiments using `payu clone` here is an example:
`payu clone -b expt -B <CONFIG_NAME> https://github.com/ACCESS-NRI/access-om3-configs <baseline_run_name>`
where the last entry is `<baseline_run_name>`, `<test_run_1_name>` etc. 

This command should be run in `<PROJECT_PATH>` when creating sensitivity experiments. 

All sensitivity experiments should have the same changes as the  `<baseline_run_name>` and the following in the `config.yaml`:
```
 env:
        ESMF_RUNTIME_PROFILE: "on"
        ESMF_RUNTIME_PROFILE_OUTPUT: "SUMMARY BINARY"
```

## MOM6 mask-table and layout guardrails for finer-resolution configs

This section is critical for MOM6/OCN PE-layout optimisation of 25 km and 8 km configurations. It is usually not needed for coarser configurations unless the existing config already uses a MOM mask table and layout pair.

MOM6 uses `MASKTABLE` and `LAYOUT` in `MOM_input`, for example:

```fortran
MASKTABLE = "mask_table.1027.72x48"
LAYOUT = 72, 48
```

The mask table and layout must match the actual MOM/OCN processor layout. If the OCN PE count or MOM layout changes, the old mask table may become invalid.

Rules:

1. During inspection, detect whether `MOM_input` contains `MASKTABLE` and `LAYOUT`.
2. Detect whether the config is a finer-resolution MOM6 config, especially global 25 km or 8 km.
3. If changing OCN/MOM PE layout for a 25 km or 8 km config, treat mask-table regeneration as mandatory unless you can prove the existing mask table is still valid.
4. Use the ACCESS-NRI `om3-scripts` repository locally if available. The relevant script is:

```text
<OM3_SCRIPTS_PATH>/masktable_generation/gen_masktable.sh
```

5. If the repo is not available locally, locate it or ask before cloning/copying. Do not silently skip mask-table handling.
6. Generate candidate mask tables using the local grid files for `ocean_hgrid.nc` and `ocean_topog.nc`.
7. Prefer placing the generated mask table inside the experiment/configuration directory, or a clearly named subdirectory under it, so the optimisation diff is self-contained and easy to review.
8. Update `MOM_input` so that:

```fortran
MASKTABLE = "<new mask table filename or relative path>"
LAYOUT = <layout_x>, <layout_y>
```

9. Update `config.yaml` or any manifest/input-file mapping so the new mask table is staged into the run directory.
10. Not all `LAYOUT = X, Y` combinations will work. Treat this as a constrained trial-and-error search, not a guaranteed algebraic substitution.
11. Prefer candidate layouts that are reasonably square / balanced, because square-ish processor domains generally perform better and often avoid pathological decompositions.
12. Before submitting a run with a new MOM layout, show:
    - old `ocn_ntasks`, `LAYOUT`, and `MASKTABLE`
    - new `ocn_ntasks`, `LAYOUT`, and generated mask-table filename
    - exact `gen_masktable.sh` command used
    - where the generated mask table was placed
    - `MOM_input` diff
    - `config.yaml` / manifest diff
    - why the candidate layout was chosen
    - expected risk and fallback plan
13. If a candidate layout fails at startup or gives a mask/layout error, mark it as `rejected` or `inconclusive`, document it in the progress Markdown, and do not keep retrying indefinitely.
14. For preliminary optimisation, test only a small number of plausible near-square layouts unless the user explicitly approves a broader mask-table/layout campaign.

Example exact-layout generation pattern:

```bash
cd <PROJECT_PATH>/profiling_analysis/masktables/<layout_x>x<layout_y>
<OM3_SCRIPTS_PATH>/masktable_generation/gen_masktable.sh \
  -g <path/to/ocean_hgrid.nc> \
  -t <path/to/ocean_topog.nc> \
  -l <layout_x> <layout_y>
```

Example range-search pattern, only if explicitly useful and affordable:

```bash
cd <PROJECT_PATH>/profiling_analysis/masktables/range_<min>_<max>
<OM3_SCRIPTS_PATH>/masktable_generation/gen_masktable.sh \
  -g <path/to/ocean_hgrid.nc> \
  -t <path/to/ocean_topog.nc> \
  -r <min_processors> <max_processors>
```

Do not confuse total `ncpus` with MOM `LAYOUT`. In coupled configs, `ncpus` may include ATM, ICE, ROF, WAV, MED, and OCN. The MOM `LAYOUT = X, Y` must correspond to the MOM/OCN decomposition, not necessarily total job CPUs.

---

## Required output structure

Create and maintain this directory structure. `PROJECT_PATH` is the top-level project directory; the baseline and all test runs are subdirectories alongside `profiling_analysis/`:

```text
<PROJECT_PATH>/
├── profiling_analysis/
│   ├── <CONFIG_NAME>_optimisation_progress.md
│   ├── <CONFIG_NAME>_performance_notes.md
│   ├── <CONFIG_NAME>_performance_summary.ipynb
│   ├── <CONFIG_NAME>_github_issue_draft.md
│   ├── run_registry.yaml
│   ├── performance_summary/
│   │   ├── run_summary.csv
│   │   ├── component_timing.csv
│   │   ├── cost_summary.csv
│   │   ├── io_summary.csv
│   │   ├── pe_layout_summary.csv
│   │   ├── mom_masktable_summary.csv
│   │   └── optimisation_summary.json
│   ├── figures/
│   │   ├── cost_improvement.png
│   │   ├── wallclock_improvement.png
│   │   ├── component_timing_critical_path.png
│   │   ├── io_bottleneck_summary.png
│   │   ├── pe_headroom.png
│   │   └── optimisation_summary_panel.png
│   └── scripts/
│       ├── parse_performance_logs.py
│       ├── update_performance_summary.py
│       └── generate_performance_notebook.py
├── <baseline_run_name>/    ← baseline payu run directory
├── <test_run_1_name>/      ← first test run directory
└── <test_run_N_name>/      ← further test run directories
```

If some files are not useful for a config, create them with a short note explaining why they are empty or not applicable.

---

## Documentation update rule after every Codex interaction

At the end of **every Codex response**, update or append to the progress Markdown before reporting back to me. This includes responses where no run was submitted.

Every Codex response should finish with a short status block like:

```markdown
## Documentation status
- Progress Markdown updated: yes/no — <path>
- Performance notes updated: yes/no — <path>
- Run registry updated: yes/no — <path>
- Summary CSV/JSON updated: yes/no — <paths>
- Notebook refreshed: yes/no — <path>
- Figures refreshed: yes/no — <paths>
- GitHub issue draft updated: yes/no — <path>
- Reason if anything was not updated: ...
```

If the notebook cannot yet be meaningfully plotted because only one run exists, still create the notebook with placeholder sections and at least the baseline table. Mark missing plots as pending in both the notebook and progress Markdown.

---

## Progress Markdown requirements

Create or update:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_optimisation_progress.md
```

This file is the running log for the optimisation thread. It must be updated:

- after initial inspection
- after existing baseline logs are parsed
- before every proposed run
- after every approved run completes
- after every rejected or abandoned optimisation idea
- after every notebook/figure refresh
- before final recommendation

Use this structure:

```markdown
# <CONFIG_NAME> optimisation progress

## Current status
- Status:
- Best accepted configuration:
- Current bottleneck:
- Next proposed action:
- Last updated:

## Safety constraints checked
| Constraint | Status | Notes |
|---|---:|---|
| Science unchanged | | |
| Output frequency unchanged | | |
| Restart frequency unchanged | | |
| Output fields unchanged | | |
| Components unchanged | | |
| Forcing paths unchanged | | |
| MOM mask table/layout consistent, if applicable | | |

## MOM mask-table status, if applicable

| Item | Value | Notes |
|---|---|---|
| Finer-resolution MOM6 config? | | |
| Current `MASKTABLE` | | |
| Current `LAYOUT` | | |
| Current OCN/MOM ntasks | | |
| Current OCN/MOM rootpe | | |
| Mask-table location in config.yaml/manifest | | |
| `om3-scripts` local path | | |
| Mask-table regeneration needed? | | |
| Candidate layouts tested | | |
| Accepted mask table | | |

## Configuration snapshot
| Item | Value |
|---|---|
| Local path | |
| Git branch | |
| Git commit | |
| Dirty state | |
| Executable | |
| Queue | |
| ncpus | |
| mem | |
| walltime | |
| run length | |
| restart frequency | |

## Active component topology
| Component | Active? | ntasks | rootpe | PET range | Notes |
|---|---:|---:|---:|---|---|
| MED/CPL | | | | | |
| ATM/DATM | | | | | |
| ROF/DROF | | | | | |
| OCN/MOM | | | | | |
| ICE/CICE | | | | | |
| WAV/WW3 | | | | | |

## Run registry summary
| Label | Output | Job ID | Purpose | Status | Accepted? | Notes |
|---|---|---|---|---|---:|---|

## Performance summary
| Label | ncpus | Sim days | Walltime | SU/day | SU/year est | Ensemble s/step | Bottleneck |
|---|---:|---:|---:|---:|---:|---:|---|

## Bottleneck evidence
Summarise the evidence from ESMF profile summaries, trace output, and component logs.

## Decision log
| Date/time | Decision | Reason | Files changed |
|---|---|---|---|

## Proposed tests
| Test | Purpose | Change | Expected benefit | Risk | Approval status |
|---|---|---|---|---|---|

## Notebook and figure status
| Artefact | Status | Notes |
|---|---:|---|
| run_summary.csv | | |
| component_timing.csv | | |
| cost_summary.csv | | |
| io_summary.csv | | |
| pe_layout_summary.csv | | |
| performance notebook | | |
| GitHub issue draft | | |
```

Keep this file concise but complete. It should allow a collaborator to understand what has happened without reading the whole Codex thread.

---

## Machine-readable summary requirements

Maintain:

```text
<PROJECT_PATH>/profiling_analysis/run_registry.yaml
<PROJECT_PATH>/profiling_analysis/performance_summary/run_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/component_timing.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/cost_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/io_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/pe_layout_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/mom_masktable_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/optimisation_summary.json
```

### `run_registry.yaml`

Each run must have an entry like:

```yaml
- label: output000_baseline
  output: output000
  job_id: null
  status: complete
  accepted: false
  purpose: baseline profiling
  local_archive_path: null
  run_start: null
  run_end: null
  simulated_days: null
  notes: ""
  changes:
    config_yaml: []
    nuopc_runconfig: []
    other: []
```

### `run_summary.csv`

Required columns:

```text
label,output,job_id,run_type,status,accepted,ncpus,mem_gb,walltime_requested,walltime_used,walltime_s,model_runtime_s,simulated_days,days_per_hour,years_per_day,git_commit,dirty_state,notes
```

### `component_timing.csv`

Required columns, using `nan` for missing/non-applicable components:

```text
label,ensemble_s_per_step,atm_s_per_step,rof_s_per_step,ocn_s_per_step,ice_s_per_step,wav_s_per_step,med_s_per_step,dominant_component,critical_path_notes
```

### `cost_summary.csv`

Required columns:

```text
label,ncpus,simulated_days,walltime_s,node_hours,cpu_hours,su_total,su_per_day,su_per_year_est,cpu_hours_per_model_year,percent_su_reduction_vs_baseline,percent_walltime_reduction_vs_baseline
```

If exact SU accounting is unavailable, estimate it transparently and mark the field or notes accordingly.

### `io_summary.csv`

Required columns:

```text
label,atm_readio_s_total,atm_readio_s_per_step,atm_readio_s_per_call,ocn_io_s_total,ice_io_s_total,wav_io_s_total,restart_read_s,restart_write_s,history_write_s,io_notes
```

Use component-specific names if DATM/DROF/PIO timers differ in this config.

### `pe_layout_summary.csv`

Required columns:

```text
label,ncpus,med_ntasks,med_rootpe,atm_ntasks,atm_rootpe,rof_ntasks,rof_rootpe,ocn_ntasks,ocn_rootpe,ice_ntasks,ice_rootpe,wav_ntasks,wav_rootpe,unused_pes,layout_notes
```

### `mom_masktable_summary.csv`

Use this file for MOM6 configs where `MASKTABLE` and `LAYOUT` are relevant. Create the file with a short `not_applicable` row for configs without MOM/OCN or without a mask-table dependency.

Required columns:

```text
label,applicable,resolution_hint,ocn_ntasks,ocn_rootpe,layout_x,layout_y,masktable,masktable_path,gen_command,hgrid_path,topog_path,status,notes
```

---

## Notebook requirements

Create and maintain:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_summary.ipynb
```

The notebook should be re-runnable and should load the CSV/JSON summaries from:

```text
<PROJECT_PATH>/profiling_analysis/performance_summary/
```

Use `matplotlib` and `pandas`. Do not use seaborn.

The notebook should follow the same style as the existing MCW performance summary notebook pattern:

1. Clear title and model/config metadata.
2. Absolute or robust relative paths.
3. Tables first, then plots.
4. Short markdown narrative before each major plot.
5. Saved figures under `<PROJECT_PATH>/profiling_analysis/figures/`.
6. A final recommendation section.
7. A “How to add future runs” section.

Required notebook sections:

```text
A. Executive summary
B. Run configuration comparison
C. Cost improvement
D. Wall-clock / runtime comparison
E. Component timing and critical path
F. I/O bottleneck summary
G. Validation of accepted candidate
H. PE headroom / idle-time interpretation
I. Optimisation summary panel
J. Final recommendation
K. How to add future runs
```

### Notebook model to follow

Follow the pattern of the provided `MCW_100km_ERA5_KPP_performance_summary.ipynb` notebook:

- title cell with model/config metadata
- a setup cell that defines `HERE`, `SUM_DIR`, and `FIG_DIR`
- load `run_summary.csv`, `component_timing.csv`, `cost_summary.csv`, and the I/O summary CSV
- define display labels in one dictionary near the top
- define plotting order from the run registry/CSV, not hard-coded plot cells
- create tables before plots
- save every plotted figure to `<PROJECT_PATH>/profiling_analysis/figures/`
- include a final section explaining how to add future runs

The notebook should be generated or refreshed by a script where practical, for example:

```text
<PROJECT_PATH>/profiling_analysis/scripts/generate_performance_notebook.py
```

The notebook must be safe to re-run from top to bottom after a new run is added. Adding a future run should normally require only:

1. add one entry to `run_registry.yaml`;
2. append rows to the summary CSVs;
3. optionally add a display label;
4. re-run the notebook.

Do not scatter manually typed performance numbers through plotting cells. If a value cannot be parsed reliably, put it in one clearly marked manual metadata block near the top.

### Required plots

Create at least these figures and save each to `<PROJECT_PATH>/profiling_analysis/figures/`:

#### 1. `cost_improvement.png`

Show:

- SU/day or CPU-hours/day for each run
- projected SU/model-year or CPU-hours/model-year
- percent reduction relative to baseline

#### 2. `wallclock_improvement.png`

Show:

- walltime/model runtime per run
- normalised walltime per fixed simulated interval, if run lengths differ
- days/hour or years/day

#### 3. `component_timing_critical_path.png`

Show grouped bars for active components, for example:

- ATM/DATM
- ROF/DROF
- OCN/MOM
- ICE/CICE
- WAV/WW3
- MED/CPL
- Ensemble / total step time

Only include components active in the inspected config.

#### 4. `io_bottleneck_summary.png`

Show relevant I/O timers, for example:

- DATM read time
- DROF read time
- restart read/write
- history output time
- PIO test results if applicable

If I/O is not the bottleneck, still include a small plot or table explaining that.

#### 5. `pe_headroom.png`

Show how close each active component is to the critical path. This should help explain whether a component has too many PEs, too few PEs, or is on the critical path.

#### 6. `optimisation_summary_panel.png`

A compact multi-panel summary suitable for a GitHub issue. It should show:

- cost before/after
- wall-clock before/after
- component timing before/after
- final recommended layout

Use clear labels and annotations. The figure should stand alone in a GitHub issue.


### Optional/special-case notebook sections

Add these sections only when relevant to the config and evidence:

```text
L. Sensitivity tests that are science-changing / not accepted
M. Run-segment or startup-overhead amortisation
N. Forcing/data-layout optimisation such as ERA5 rechunking
O. Longer validation run / production validation
```

Important: science-changing sensitivity results may be plotted, but must be visually and textually labelled as **not accepted production changes**. Accepted no-science-change changes must remain separate from sensitivity/future-work sections.

For an ERA5/DATM forcing-layout investigation, include plots such as:

- DATM read time per call before/after
- ATM or DATM seconds per step before/after
- ensemble seconds per step before/after
- SU/model-year before/after
- bottleneck migration before/after

---

## GitHub issue draft requirements

Create/update:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_github_issue_draft.md
```

The issue draft should be updated whenever the notebook is updated.

Use this structure:

```markdown
# Performance optimisation summary for <CONFIG_NAME>

## Summary

## Baseline problem

## Method

## Runs compared

## Key results

## Figures

### Cost improvement
![Cost improvement](<PROJECT_PATH>/profiling_analysis/figures/cost_improvement.png)

### Wall-clock improvement
![Wall-clock improvement](<PROJECT_PATH>/profiling_analysis/figures/wallclock_improvement.png)

### Component timing / critical path
![Component timing](<PROJECT_PATH>/profiling_analysis/figures/component_timing_critical_path.png)

### I/O bottleneck
![I/O bottleneck](<PROJECT_PATH>/profiling_analysis/figures/io_bottleneck_summary.png)

### PE headroom
![PE headroom](<PROJECT_PATH>/profiling_analysis/figures/pe_headroom.png)

### Optimisation summary
![Optimisation summary](<PROJECT_PATH>/profiling_analysis/figures/optimisation_summary_panel.png)

## Safety checks

| Check | Status |
|---|---:|
| Science unchanged | |
| Output frequency unchanged | |
| Restart frequency unchanged | |
| Output fields unchanged | |
| Active components unchanged | |
| Forcing paths unchanged | |

## Recommended configuration

## Remaining limitations

## Next steps
```

The issue draft must clearly distinguish:

- accepted no-science-change optimisations
- rejected tests
- inconclusive tests
- tests that would be science-changing
- future work outside the current preliminary pass

---

## Stage 1 — Inspect the config

Start with:

```bash
cd <LOCAL_CONFIG_PATH>
git status
git branch --show-current
git rev-parse HEAD
```

Inspect at minimum, if present:

```text
config.yaml
nuopc.runseq
nuopc.runconfig
drv_in
input.nml
MOM_input
MOM_override
ice_in
wav_in
datm_in
drof_in
datm.streams.xml
drof.streams.xml
manifests/
payu config metadata
existing archive/work/log directories
```

Record in the progress Markdown:

- branch and commit
- dirty state
- executable
- queue, ncpus, walltime, mem, jobfs
- active components
- PE layout
- run sequence
- coupling frequency
- run length
- restart settings
- output settings
- current profiling settings
- MOM `MASKTABLE` and `LAYOUT`, if `MOM_input` exists
- whether the config appears to be 25 km, 8 km, or another finer-resolution MOM6 case
- whether `om3-scripts` and `masktable_generation/gen_masktable.sh` are available locally

Do not change anything in this stage.

---

## Stage 2 — Gather existing baseline timing

Before proposing any run, search existing archive/work/logs for completed runs of this config.

Collect, if available:

- Payu timing JSON
- PBS stdout/stderr
- ESMF profile summary
- PET logs, if any
- `med.log`
- component logs
- MOM timing
- CICE timing
- WW3 timing
- DATM/DROF stream read timings
- restart/history I/O timing
- esmf-trace output, if any

Use `access-profiling` and `esmf-trace` if they are available and useful. If they are not immediately usable, write a small reproducible parser instead of spending too long fighting the environment.

Update:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_optimisation_progress.md
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_notes.md
<PROJECT_PATH>/profiling_analysis/run_registry.yaml
<PROJECT_PATH>/profiling_analysis/performance_summary/*.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/optimisation_summary.json
```

Then create or refresh the notebook, even if it initially contains only baseline data.

---

## Stage 3 — Create the baseline notebook skeleton

Before any new test run, create a first version of:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_summary.ipynb
```

At this stage it may contain only baseline data, but it must already include:

- run configuration table
- baseline timing table
- cost estimate table
- component timing plot, if component data exists
- a placeholder section explaining which plots need more runs
- “How to add future runs”

Also create:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_github_issue_draft.md
```

It can be marked as a draft / incomplete until at least one optimisation test exists.

---

## Stage 4 — Diagnose bottleneck and propose first test family

Use profile evidence to identify the first optimisation family. Possible families include:

- PE rebalancing / resource layout
- component rank redistribution
- I/O/Pio tuning
- run-segment amortisation
- restart/history I/O investigation
- forcing file layout investigation
- load imbalance within a component
- queue/resource mismatch

Do not assume the bottleneck. For example:

- A MOM6-CICE6 config may bottleneck on OCN, ICE, ATM, ROF, MED, or I/O.
- A MOM6-CICE6-WW3 config may bottleneck on OCN, ICE, WAV, ATM/DATM, ROF/DROF, MED, or waiting/load imbalance.
- A standalone WW3 config may bottleneck on WAV, DATM, DICE/ICE, MED, ROF, I/O, or startup.

Write the diagnosis into the progress Markdown and performance notes.


### Extra rule for MOM/OCN PE-layout tests on 25 km and 8 km configs

If the proposed first test family changes OCN/MOM PE layout for a finer-resolution MOM6 config, include mask-table handling in the proposal. Do not propose only `ocn_ntasks` changes. The proposal must include:

- candidate `LAYOUT = X, Y` values
- why each candidate is near-square or otherwise reasonable
- whether the old mask table is invalidated
- the exact `gen_masktable.sh` command for each candidate
- where generated mask tables will be stored
- how `MOM_input` and `config.yaml`/manifest will be updated
- how startup failure or invalid mask/layout combinations will be documented

Keep this preliminary pass small. Prefer 1–3 plausible candidate layouts rather than a broad trial-and-error campaign unless the user approves a larger sweep.

Then propose at most one first test family.

Before any run, show:

1. Existing baseline timing summary.
2. Bottleneck evidence.
3. Exact proposed test.
4. Expected benefit.
5. Risk level.
6. Exact files to change.
7. Full `git diff`.
8. Confirmation that science/output/restart/components/forcing are unchanged.
9. Exact run command.
10. Which progress/CSV/notebook files will be updated after the run.

Stop and ask for approval before submitting.

---

## Stage 5 — Run approval gate

Do not run unless I explicitly approve.

When asking for approval, use this format:

```markdown
## Approval request

I propose to run: <test label>

### Why
...

### Files changed
...

### Safety check
| Check | Status |
|---|---:|
| Science unchanged | ✅ |
| Output frequency unchanged | ✅ |
| Restart frequency unchanged | ✅ |
| Output fields unchanged | ✅ |
| Active components unchanged | ✅ |
| Forcing paths unchanged | ✅ |
| MOM mask table/layout consistent, if applicable | ✅ |

### Expected result
...

### Exact command
```bash
<command>
```

Please approve before I submit.
```

---

## Stage 6 — After every completed run

After each completed run, immediately update all of:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_optimisation_progress.md
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_notes.md
<PROJECT_PATH>/profiling_analysis/run_registry.yaml
<PROJECT_PATH>/profiling_analysis/performance_summary/run_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/component_timing.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/cost_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/io_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/pe_layout_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/mom_masktable_summary.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/optimisation_summary.json
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_summary.ipynb
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_github_issue_draft.md
<PROJECT_PATH>/profiling_analysis/figures/*.png
```

For the completed run, report:

- job ID
- output directory
- run interval
- simulated days
- walltime
- model runtime
- CPU-hours
- SU total, if available
- SU/day
- projected SU/year or CPU-hours/model-year
- memory used
- active PE layout
- MOM `MASKTABLE` and `LAYOUT`, if applicable
- generated mask-table path/status, if applicable
- component timings
- dominant bottleneck
- whether the bottleneck moved
- whether the test is accepted, rejected, or inconclusive
- whether a follow-up test is justified

Then refresh the notebook and figures. The notebook must be useful even after only baseline + one test.

---

## Stage 7 — Preliminary test budget

Do not run an unlimited optimisation campaign.

Use this policy:

1. Baseline/profile run if existing data is insufficient.
2. Up to `MAX_NEW_TEST_RUNS_FOR_PRELIMINARY_PASS` optimisation tests.
3. After each test, decide whether the next test is still justified.
4. Stop early if:
   - the improvement is negligible,
   - the bottleneck is not affected,
   - the next test would be science-changing,
   - cost/wall-clock is already acceptable,
   - uncertainty/noise is larger than the observed improvement.

If many tests are needed, propose a separate follow-up campaign rather than continuing inside the preliminary pass.

---

## Stage 8 — Test interpretation rules

Classify every test as one of:

```text
accepted
rejected
inconclusive
diagnostic-only
science-changing / not accepted
future-work
```

Use this decision logic:

- Accept if it improves the chosen objective and passes safety checks.
- Reject if it worsens the objective, introduces risk, or has no meaningful benefit.
- Mark inconclusive if run-to-run variability is comparable to the measured change.
- Mark science-changing if it changes physics/science settings, even if faster.
- Mark future-work if it requires data preprocessing, code changes, forcing restructuring, or a larger campaign.

Always separate:

- performance-only changes
- resource-only changes
- I/O-only changes
- science-changing sensitivity tests
- forcing-data restructuring tests

---

## Stage 9 — Notebook visual style

Use a clean style suitable for GitHub issue screenshots.

Notebook plotting requirements:

- `matplotlib` only.
- Save figures as PNG.
- Use readable labels.
- Annotate key percent changes.
- Use consistent run labels.
- Use clear titles.
- Use concise markdown explanation before each figure.
- Avoid relying on hard-coded single-case component names. Detect active components and gracefully skip inactive ones.
- If run lengths differ, show both raw and normalised metrics.
- If exact SU is unavailable, show CPU-hours/model-year and state that SU is estimated or unavailable.

The notebook should be robust to future runs: adding a new row to CSVs and `run_registry.yaml` should be enough to update plots.

---

## Stage 10 — Final recommendation

At the end of the preliminary pass, update:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_optimisation_progress.md
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_notes.md
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_summary.ipynb
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_github_issue_draft.md
```

The final recommendation must include:

1. Accepted configuration changes.
2. Rejected changes and why.
3. Best cost-efficient option.
4. Best wall-clock option, if different.
5. Estimated improvement:
   - wall-clock
   - days/hour or years/day
   - CPU-hours/model-year or SU/year
6. Remaining bottleneck.
7. Safety checks.
8. Suggested future work.
9. MOM mask-table/layout status, if applicable.
10. Exact final `git diff`.
11. Whether a final validation run is recommended.

The final notebook must contain the complete story in plots and tables.

---

## Stage 11 — Start now

Start by inspecting the local config and existing logs only.

Do not submit a job yet.

Your first response should include:

1. What files and logs you inspected.
2. The active component topology.
3. Existing baseline timing evidence.
4. MOM mask-table/layout status, if applicable.
5. Whether `access-profiling` worked.
6. Whether `esmf-trace` worked.
7. The initial progress Markdown path.
8. The initial notebook path.
9. The initial GitHub issue draft path.
10. What information is still missing.
11. The proposed next action, if any.

If a profiling or optimisation run is needed, stop at the approval gate before submitting.
