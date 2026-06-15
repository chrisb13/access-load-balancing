---
name: feedback-access-om3-short-runs
description: How to correctly set run length and restart period for short ACCESS-OM3 test runs
metadata:
  type: feedback
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

For short ACCESS-OM3 test runs, set BOTH the stop AND restart parameters in `nuopc.runconfig` to match the desired run length.

**Why:** The model only writes restarts at intervals defined by `restart_n`/`restart_option`. If these remain at the default `1 nyears` but the run is only 35 days, no restart is ever written and payu fails at archiving with "Restart pointer file not found: rpointer.cpl".

**How to apply:** In `nuopc.runconfig` under `CLOCK_attributes`, set restart to match stop:

```
stop_n          = 35
stop_option     = ndays
restart_n       = 35
restart_option  = ndays
```

Docs: https://docs.access-hive.org.au/models/run_a_model/run_access-om3/#change-run-length-and-restart-period

Also set `restart_freq: 35D` in `config.yaml` (payu archiving frequency). Setting only `restart_freq` in config.yaml is NOT sufficient — it does not propagate to the model's restart writing frequency in payu 1.3.2.
