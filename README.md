# Universal ACCESS-OM3 Preliminary Profiling + Optimisation Prompt for Codex/Claude
**Authors:** Ezhilsabareesh Kannadasan, Christopher Bull.
**Version:** v0.2  
**Purpose:** A reusable single-prompt workflow for preliminary optimisation of any ACCESS-OM3-style Payu configuration, with **continuous progress tracking after every Codex interaction**, **machine-readable performance summaries**, a **shareable, re-runnable Jupyter notebook with plots**, and **MOM6 mask-table/layout safety for finer-resolution configs**.

This version is designed for configs such as:

- MOM6-CICE6
- MOM6-CICE6-WW3
- other ACCESS-OM3 / NUOPC / Payu configurations

It is intentionally component-agnostic. Codex must infer the active components from the local configuration, not assume that WW3/WAV, MOM/OCN, CICE/ICE, DATM/ATM, DROF/ROF, or MED are always present.

## How to use this repository

The below assumes you have one run completed that is inside your `PROJECT_PATH` (e.g. `/g/data/tm70/cyb561/access-profiling`) that has the following addition in the `config.yaml`:
```
 env:
        ESMF_RUNTIME_PROFILE: "on"
        ESMF_RUNTIME_PROFILE_OUTPUT: "SUMMARY BINARY"
```

You paste `AGENTS.md` into a fresh AI thread, fill in the Case input block (config name, local paths), and the AI then drives itself through 11 structured "stages":

1.	Inspect config (read-only)
   
2.	Gather existing baseline timing from logs
  
3.	Create baseline Jupyter notebook skeleton
   
4.	Diagnose bottleneck, propose first test
   
5.	Approval gate — stops for human sign-off before any job submission
    
6.	After each run: update all CSVs, notebook, figures
    
7-8.	Budget & interpretation (max 3 test runs)

9.	Notebook visual style (matplotlib, GitHub-ready)
    
10. Final recommendation with full git diff
    
11 Go time!

