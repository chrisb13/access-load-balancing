# Universal ACCESS-OM3 Preliminary Profiling + Optimisation Prompt for Codex/Claude
**Authors:** Ezhilsabareesh Kannadasan, Christopher Bull
**Version:** v0.2  
**Purpose:** A reusable single-prompt workflow for preliminary optimisation of any ACCESS-OM3-style Payu configuration, with **continuous progress tracking after every Codex interaction**, **machine-readable performance summaries**, a **shareable, re-runnable Jupyter notebook with plots**, and **MOM6 mask-table/layout safety for finer-resolution configs**.

This version is designed for configs such as:

- MOM6-CICE6
- MOM6-CICE6-WW3
- other ACCESS-OM3 / NUOPC / Payu configurations

It is intentionally component-agnostic. Codex must infer the active components from the local configuration, not assume that WW3/WAV, MOM/OCN, CICE/ICE, DATM/ATM, DROF/ROF, or MED are always present.

---

## How to use this file

Copy the entire prompt `AGENTS.md` into a fresh Codex (or other AI tool) thread.

Before sending it, replace only the values in the **Case input block**. Leave the rest unchanged.

