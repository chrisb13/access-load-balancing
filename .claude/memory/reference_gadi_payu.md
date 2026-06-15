---
name: reference-gadi-payu
description: How to load payu on Gadi for non-interactive SSH sessions
metadata:
  type: reference
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

`payu` is not in the default SSH PATH on Gadi. Load it with:

```bash
module purge
module use /g/data/vk83/modules
module load payu
```

Then use `payu sweep` and `payu run` as normal.
