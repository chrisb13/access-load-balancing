---
name: reference-gadi-qstat
description: qstat is not in default PATH on Gadi login nodes via SSH; use full path
metadata: 
  node_type: memory
  type: reference
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

`qstat` is not in the default PATH when SSHing to Gadi non-interactively. Use the full path:

```bash
/opt/nci/pbs/2024.1.2.20241017100211/bin/qstat -u <username>
```

Found via: `find /usr /opt /PBS -name qstat 2>/dev/null`

Multiple PBS versions exist under `/opt/nci/pbs/`; use the most recent one (2024.x).

Useful flags:
- `qstat -u cyb561` — list all jobs for user
- `qstat -xf <jobid>` — full details of a completed/running job (including storage flags)
