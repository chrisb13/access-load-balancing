---
name: feedback-gadi-normalsr-walltime
description: normalsr queue on Gadi has a 5-hour walltime cap for large jobs (e.g. 4992 CPUs); do not request more than 05:00:00
metadata:
  type: feedback
---

The `normalsr` queue on Gadi enforces a **maximum walltime of 5 hours** for jobs using large core counts (confirmed at 4992 CPUs / 48 nodes). Requesting 6h raises `ValueError: Requested walltime of 6.0 hours exceeds the limit of 5 hours for queue 'normalsr'`.

**Why:** Hit this when setting up 8km ACCESS-OM3 sensitivity tests; the slow end of the partition sweep (fewer OCN cores) was estimated to need ~5-6h but the queue cap is hard at 5h.

**How to apply:** Always cap `walltime` at `05:00:00` for `normalsr` jobs regardless of estimated runtime. If a run is expected to exceed 5h, either reduce the run length or switch to a queue with a higher walltime limit (e.g. `normalbw` or `hugemem`). For 8km ACCESS-OM3 profiling runs, 32 simulated days at 4992 cores fits within 5h across all tested OCN/non-OCN partitions.
