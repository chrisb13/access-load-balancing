# Prompt to send to AI tool

You are helping perform a preliminary performance profiling and optimisation pass for an ACCESS-OM3 / MOM6-CICE6 / MOM6-CICE6-WW3 / standalone-component configuration on NCI/Gadi.

This is a **single-prompt, multi-gate workflow**. Do not wait for me to provide a new prompt after every stage. Continue through the stages below, but stop at each approval gate before changing model configuration in a risky way or submitting jobs.

The goal is not blind tuning. The goal is to use ESMF profile summaries, Payu/PBS logs, component logs, and, where useful, `access-profiling` / `esmf-trace` to identify the first safe optimisation family for this config.

**Core objective**: the goal is to reduce the overall runtime of the model while keeping model cost reasonably low. The primary optimisation pathway is a nested node/partition search:

1. For each candidate node count, keep the total core count fixed as `ncpus = nodes * CORES_PER_NODE`.
2. At that fixed total core count, change the partition between ocean and non-ocean model components.
3. Repeat the partition search across selected node counts between `MIN_NODES` and `MAX_NODES`, starting with the median node count.

In `nuopc.runconfig`, the non-ocean model components are `atm_ntasks`, `cpl_ntasks`, `ice_ntasks`, `ocn_rootpe`, and `rof_ntasks`; these should all be set to the same `non_ocn_ntasks` value. The ocean component is `ocn_ntasks`, with `ocn_rootpe = non_ocn_ntasks` and `ocn_ntasks = ncpus - non_ocn_ntasks`.

Very important: this workflow must produce and keep updating both:

1. A **progress Markdown file**, updated after every meaningful step and **after every completed run**.
2. A **re-runnable performance summary notebook**, with plots, tables, and a GitHub-issue-ready optimisation story.

Do not leave plotting and documentation until the end. Start the structure early, then keep it updated after every meaningful Codex response, every approved/rejected idea, and every completed run.

Wider Context. Here, we are trying to optimise the global 8km configuration (ocean and sea ice no biogeochemistry), here are the below similar performance numbers for the last generation of the model (ACCESS-OM2):

| Model              | Res.           | Processors\* | cost (kSU/yr) | Walltime (yrs/day) |
| ------------------ | -------------- | ------------ | ------------- | ------------------ |
| ACCESS-OM2-01      | 0.1°,75 levels | 5088         | 123           | 2                  |
| ACCESS-OM2-01-BGC† | 0.1°,75 levels | 5088         | 245           | 1                  |
| ACCESS-OM2-01      | 0.1°,75 levels | 12144        | 156           | 3.7                |
| ACCESS-OM2-01-BGC† | 0.1°,75 levels | 12144        | 310           | 1.9

---

## Case input block

Use these values for this optimisation thread:

```text
CONFIG_NAME = "<replace with config name, e.g. dev-MC_25km_jra_iaf or MCW_100km_era5_iaf_KPP>"

GITHUB_BRANCH_URL = "<replace with branch URL, e.g. https://github.com/ACCESS-NRI/access-om3-configs/tree/dev-MC_25km_jra_iaf>"

BASE_CONFIG_PATH = "<replace with Gadi path to the source/reference config checkout used to create the baseline experiment>"

PROJECT_PATH = "<replace with top-level project directory; the baseline run, all test runs, and profiling_analysis/ all reside directly inside here>"

# Baseline/control setup. By default, the workflow creates the baseline/control experiment instead of assuming it already exists.
BASELINE_SETUP_MODE = "create-baseline"  # options: create-baseline, existing-baseline

BASELINE_RUN_NAME = "<replace with baseline/control experiment name>"

EXISTING_BASELINE_PATH = "<only fill this if BASELINE_SETUP_MODE = existing-baseline>"

# Experiment setup backend. Use access-experiment-generator by default for baseline + sweep/ensemble setup.
USE_EXPERIMENT_GENERATOR = "yes"  # options: yes, no

EXPERIMENT_GENERATOR_MODULE_COMMAND = "module use /g/data/vk83/prerelease/modules && module load payu/dev"

EXPERIMENT_GENERATOR_YAML = "<PROJECT_PATH>/profiling_analysis/Experiment_generator_<CONFIG_NAME>.yaml"

EXPERIMENT_SETUP_POLICY = "If USE_EXPERIMENT_GENERATOR = yes, use access-experiment-generator to create the baseline/control experiment and all approved test experiments from a YAML plan. If USE_EXPERIMENT_GENERATOR = no, use payu clone/manual setup and document why."

ESMF_TRACE_PATH = "<replace with path to esmf-trace tool, or leave blank if not available>"

EXPECTED_RUN_COMMAND = "inspect first; usually: module use /g/data/vk83/modules && module load payu && payu run"

OPTIMISATION_OBJECTIVE = "preliminary optimisation using ESMF profile summary evidence; optimise both wall-clock and CPU-hours/model-year, but prioritise safe cost reduction if wall-clock is not harmed much"

MAX_NEW_TEST_RUNS_FOR_PRELIMINARY_PASS = 3

CORES_PER_NODE = 104

MIN_NODES = "<minimum number of Gadi nodes to test>"

MAX_NODES = "<maximum number of Gadi nodes to test>"

NODE_SEARCH_STRATEGY = "start at the median of MIN_NODES and MAX_NODES, then test lower/higher node counts only if justified by profile evidence"

PARTITION_SEARCH_STRATEGY = "for each fixed node count, keep ncpus = nodes * CORES_PER_NODE fixed and vary the ocn/non-ocn partition"

MAX_PARTITIONS_PER_NODE = 3

MAX_NODE_COUNTS_FOR_PRELIMINARY_PASS = 3

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
17. For read-only Gadi queries, batch commands to reduce approval prompts. The Stage 5 job-submission approval gate is non-negotiable.
18. When testing a new node count, first preserve the baseline ocean/non-ocean ratio unless the ESMF profile evidence clearly supports a different starting partition.
19. As we progress through the 11 stages, announce to the user which stage we are in.
20. Do not change run sequence (`nuopc.runseq`).
21. For each candidate node count, `ncpus` in `config.yaml` must equal `nodes * CORES_PER_NODE`, and `ncpus` must also equal `non_ocn_ntasks + ocn_ntasks` in `nuopc.runconfig`.
22. Start the node-count search with the median of `MIN_NODES` and `MAX_NODES`.
23. At each fixed node count, optimise the concurrent component partition by changing `non_ocn_ntasks` and `ocn_ntasks`, while keeping `ncpus` fixed. Estimate the approximate work per component from seconds/model-step, keep the estimate in a small table/array, and use it to choose the next partition.
24. After finding the best approved partition for one node count, move to lower or higher node counts between `MIN_NODES` and `MAX_NODES` only if the timing/cost evidence justifies it. Do not blindly run all combinations.
25. For components that use a layout, especially MOM6/OCN, the valid number of assignable cores must be determined from the landmasking/mask-table process. For MOM6 mask-table configs, verify that `ocn_ntasks = layout_x * layout_y - n_mask`; do not assume the first number in the mask-table filename is `ocn_ntasks`.
26. The workflow should set up the baseline/control experiment by default. Do not assume a manually created baseline already exists unless `BASELINE_SETUP_MODE = existing-baseline` and `EXISTING_BASELINE_PATH` is provided.
27. If `USE_EXPERIMENT_GENERATOR = yes`, use `access-experiment-generator` for both the baseline/control experiment and approved node/partition test experiments. Do not search for arbitrary local installations; use `EXPERIMENT_GENERATOR_MODULE_COMMAND`.
28. Do not run `experiment-generator` without explicit approval. Creating or editing the YAML plan is allowed as setup/inspection; generating experiments is a write action and must stop at the approval gate.
29. If `USE_EXPERIMENT_GENERATOR = no`, use `payu clone`/manual setup and document the reason. Every generated experiment must be traceable to either an `Experiment_generator` YAML plan or a documented fallback setup command.
30. The baseline/control experiment and all test experiments must include this profiling environment in `config.yaml`:

```yaml
env:
  ESMF_RUNTIME_PROFILE: "on"
  ESMF_RUNTIME_PROFILE_OUTPUT: "SUMMARY BINARY"
```

---

## Creating OM3 experiments

The workflow should normally create the baseline/control experiment itself. Do not assume a manually created baseline already exists unless `BASELINE_SETUP_MODE = existing-baseline`.

Use this setup decision:

```text
if BASELINE_SETUP_MODE = create-baseline:
    create the baseline/control experiment before the baseline profiling run
else if BASELINE_SETUP_MODE = existing-baseline:
    inspect EXISTING_BASELINE_PATH and verify it has the required profiling env

if USE_EXPERIMENT_GENERATOR = yes:
    use access-experiment-generator for baseline/control and approved test experiments
else:
    use payu clone/manual setup and document why
```

### Required profiling environment

The baseline/control experiment and every perturbation/test experiment must include this in `config.yaml`:

```yaml
env:
  ESMF_RUNTIME_PROFILE: "on"
  ESMF_RUNTIME_PROFILE_OUTPUT: "SUMMARY BINARY"
```

If `BASELINE_SETUP_MODE = existing-baseline` and the existing baseline does not contain this environment block, propose the minimal edit and stop for approval before running or re-running the baseline.

### Preferred experiment-generator workflow

When `USE_EXPERIMENT_GENERATOR = yes`, use `access-experiment-generator` through the known Gadi module command:

```bash
<EXPERIMENT_GENERATOR_MODULE_COMMAND>
```

Do not search for arbitrary local installations of `experiment-generator`. The expected executable is the one provided by the module. A simple `experiment-generator --help` check is allowed, but the workflow should not branch into local discovery logic.

Create or update:

```text
<PROJECT_PATH>/profiling_analysis/Experiment_generator_<CONFIG_NAME>.yaml
```

The YAML must describe:

- the source repository and start point;
- the control/baseline experiment named `BASELINE_RUN_NAME`;
- the required `config.yaml` profiling environment using `SUMMARY BINARY`;
- one branch/experiment per approved node/partition candidate;
- the `config.yaml` resource changes;
- the `nuopc.runconfig` partition changes;
- any `MOM_input`, mask-table, or manifest/config updates required for MOM6 layouts.

If a candidate changes MOM6/OCN layout for 25 km or 8 km configs, generate/validate the required mask table before finalising the YAML, then wire the generated mask-table file and matching `MOM_input`/`config.yaml` edits into the generator plan.

Before running the generator, stop and show:

1. the full experiment-generator YAML;
2. the planned control/baseline experiment name;
3. the planned test experiment names;
4. the exact files each experiment will change;
5. the generated mask-table paths, if applicable;
6. the expected directory layout under `<PROJECT_PATH>`;
7. the exact command to run;
8. the fallback/manual setup plan only if `USE_EXPERIMENT_GENERATOR = no`.

Do not run this command until explicitly approved:

```bash
cd <PROJECT_PATH>
<EXPERIMENT_GENERATOR_MODULE_COMMAND>
experiment-generator -i <PROJECT_PATH>/profiling_analysis/Experiment_generator_<CONFIG_NAME>.yaml
```

After generation, inspect and record:

- generated branches/experiment directories;
- `git diff` from the control branch for each candidate;
- whether all expected files changed and no unexpected files changed;
- the setup backend and YAML path in `run_registry.yaml`;
- the generator YAML in the progress Markdown.

### Fallback payu-clone workflow

Use this only when `USE_EXPERIMENT_GENERATOR = no` or when the generator is explicitly unsuitable for the case. Document the reason before using the fallback.

Example baseline/control setup:

```bash
cd <PROJECT_PATH>
payu clone -b expt -B <CONFIG_NAME> https://github.com/ACCESS-NRI/access-om3-configs <BASELINE_RUN_NAME>
```

For each fallback experiment, record:

- exact clone command;
- source branch/commit;
- experiment directory;
- manual edits applied;
- `git diff`;
- why experiment-generator was not used.

Do not mix experiment-generator and fallback setup for the same sweep unless there is a clear reason and it is documented.

## Node and partition search strategy

This workflow uses a nested search, not a single flat sweep.

### Outer loop: node-count search

Candidate total core counts are determined by:

```text
ncpus = nodes * CORES_PER_NODE
```

Start with the median of `MIN_NODES` and `MAX_NODES`. After evaluating the median node count, test lower or higher node counts only if the performance/cost evidence justifies it. Do not blindly run every node count unless explicitly approved.

### Inner loop: ocean/non-ocean partition search

For each fixed `ncpus`, vary the partition between ocean and non-ocean components while keeping total cores fixed.

Use this relationship:

```text
non_ocn_ntasks = atm_ntasks = cpl_ntasks = ice_ntasks = rof_ntasks = ocn_rootpe
ocn_ntasks = ncpus - non_ocn_ntasks
non_ocn_ntasks + ocn_ntasks = ncpus
```

For each proposed partition, use ESMF profile timings to estimate whether the ocean or non-ocean shared component block is on the critical path. Keep a small machine-readable table of the estimate, including:

```text
nodes,ncpus,non_ocn_ntasks,ocn_ntasks,estimated_non_ocn_s_per_step,estimated_ocn_s_per_step,expected_bottleneck,reason
```

The search order is:

1. Inspect baseline timing and PE layout.
2. Select the median node count.
3. At that fixed node count, propose a small number of ocean/non-ocean partitions.
4. Run only approved candidates.
5. Pick the best partition for that node count.
6. Move to lower or higher node counts only if justified by profile evidence.

For preliminary optimisation, limit the search using `MAX_PARTITIONS_PER_NODE` and `MAX_NODE_COUNTS_FOR_PRELIMINARY_PASS` unless the user explicitly approves a broader campaign.


## MOM6 mask-table and layout guardrails for finer-resolution configs

This section is critical for MOM6/OCN PE-layout optimisation of 25 km and 8 km configurations. It is usually not needed for coarser configurations unless the existing config already uses a MOM mask table and layout pair.

MOM6 uses `MASKTABLE` and `LAYOUT` in `MOM_input`, for example:

```fortran
MASKTABLE = "mask_table.1027.72x48"
LAYOUT = 72, 48
```

The mask table and layout must match the actual MOM/OCN processor layout. If the OCN PE count or MOM layout changes, the old mask table may become invalid.

For MOM6 mask-table configs, verify the relationship using the contents of the mask table:

```text
ocn_ntasks = layout_x * layout_y - n_mask
```

Do not assume the first number in a filename such as `mask_table.1027.72x48` is `ocn_ntasks`; it may refer to the number of masked land-only regions (`n_mask`).

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
│   ├── Experiment_generator_<CONFIG_NAME>.yaml       ← if experiment-generator is used/proposed
│   ├── experiment_setup_status.md                    ← records generator availability/fallback decisions
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

## Experiment setup status
| Item | Value | Notes |
|---|---|---|
| Setup backend | | `experiment-generator`, `payu-clone-fallback`, or `existing-experiment` |
| Baseline setup mode | | `create-baseline` or `existing-baseline` |
| Baseline/control experiment | | |
| Existing baseline path, if used | | |
| Use experiment-generator? | | `yes` or `no` |
| Experiment-generator module command | | |
| Experiment-generator YAML | | |
| Fallback reason, if used | | |
| Generated experiment branches/directories | | |

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
| Base config path | |
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
<PROJECT_PATH>/profiling_analysis/experiment_setup_status.md
<PROJECT_PATH>/profiling_analysis/Experiment_generator_<CONFIG_NAME>.yaml, if experiment-generator is used/proposed
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
  setup_backend: existing-experiment
  setup_plan_yaml: null
  setup_command: null
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
cd <BASE_CONFIG_PATH>
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
- whether `BASELINE_SETUP_MODE` is `create-baseline` or `existing-baseline`
- whether `USE_EXPERIMENT_GENERATOR` is `yes` or `no`
- the baseline/control experiment name or existing baseline path
- the experiment setup backend: `experiment-generator`, `payu-clone-fallback`, or `existing-experiment`

Do not change anything in this stage.

---

## Stage 2 — Set up or inspect the baseline/control experiment

Do not assume the baseline/control experiment already exists. Follow `BASELINE_SETUP_MODE`.

### If `BASELINE_SETUP_MODE = create-baseline`

Prepare the baseline/control experiment before any optimisation tests.

If `USE_EXPERIMENT_GENERATOR = yes`, create or update the experiment-generator YAML so that it includes a control/baseline experiment named `BASELINE_RUN_NAME` and the required profiling environment:

```yaml
env:
  ESMF_RUNTIME_PROFILE: "on"
  ESMF_RUNTIME_PROFILE_OUTPUT: "SUMMARY BINARY"
```

Stop for approval before running `experiment-generator`. After the baseline/control experiment is generated, inspect the generated config and show the diff relative to the source/start point.

If `USE_EXPERIMENT_GENERATOR = no`, prepare the baseline/control experiment using `payu clone` or another documented manual setup path. Stop for approval before creating the experiment.

### If `BASELINE_SETUP_MODE = existing-baseline`

Inspect `EXISTING_BASELINE_PATH`. Verify that the existing baseline has:

- the expected branch/commit or clearly documented provenance;
- the required profiling environment in `config.yaml`;
- no unexpected science/output/restart changes;
- usable logs, archives, or a clear reason why a baseline profiling run is still needed.

If the existing baseline is missing the required profiling env, propose the minimal edit and stop for approval before running or re-running it.

### Gather baseline timing

After the baseline/control experiment exists, search its archive/work/logs for completed runs.

Collect, if available:

- Payu timing JSON;
- PBS stdout/stderr;
- ESMF profile summary;
- ESMF binary profile output, if present;
- PET logs, if any;
- `med.log`;
- component logs;
- MOM timing;
- CICE timing;
- WW3 timing;
- DATM/DROF stream read timings;
- restart/history I/O timing;
- esmf-trace output, if any.

Use `access-profiling` and `esmf-trace` if they are available and useful. If they are not immediately usable, write a small reproducible parser instead of spending too long fighting the environment.

Update:

```text
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_optimisation_progress.md
<PROJECT_PATH>/profiling_analysis/<CONFIG_NAME>_performance_notes.md
<PROJECT_PATH>/profiling_analysis/run_registry.yaml
<PROJECT_PATH>/profiling_analysis/experiment_setup_status.md
<PROJECT_PATH>/profiling_analysis/Experiment_generator_<CONFIG_NAME>.yaml, if experiment-generator is used/proposed
<PROJECT_PATH>/profiling_analysis/performance_summary/*.csv
<PROJECT_PATH>/profiling_analysis/performance_summary/optimisation_summary.json
```

Then create or refresh the notebook, even if it initially contains only baseline data.

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
11. Experiment setup backend for this run: `experiment-generator`, `payu-clone-fallback`, or `existing-experiment`.
12. If using `experiment-generator`, the full YAML plan and exact `experiment-generator -i ...` command.
13. If `USE_EXPERIMENT_GENERATOR = no`, the documented reason and fallback setup command.

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

### Experiment setup
- Setup backend:
- Experiment-generator YAML, if used:
- Fallback reason and command, if `USE_EXPERIMENT_GENERATOR = no`:
- Experiment setup command, if separate from run command:

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
<PROJECT_PATH>/profiling_analysis/experiment_setup_status.md
<PROJECT_PATH>/profiling_analysis/Experiment_generator_<CONFIG_NAME>.yaml, if experiment-generator is used/proposed
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
- experiment setup backend and generator YAML/fallback command, if applicable
- whether this was the baseline/control run or an optimisation test
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

1. Baseline/control setup and baseline/profile run if existing data is insufficient.
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
10. Baseline/control setup provenance, including experiment-generator YAML or fallback command.
11. Exact final `git diff`.
12. Whether a final validation run is recommended.

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
7. Whether the workflow will create the baseline/control experiment or use an existing baseline.
8. Whether `USE_EXPERIMENT_GENERATOR` is `yes` or `no`, and which experiment setup backend will be used.
9. The initial progress Markdown path.
10. The initial notebook path.
11. The initial GitHub issue draft path.
12. What information is still missing.
13. The proposed next action, if any.

If a profiling or optimisation run is needed, stop at the approval gate before submitting.