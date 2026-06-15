---
name: feedback-payu-sweep
description: "After a failed payu job, always run `payu sweep` before `payu run` to clean up the failed state"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

After a payu job fails, always run `payu sweep` before `payu run` to clean up the work directory and reset payu's state.

**Why:** payu leaves behind lock files and work directory state from the failed run; `payu run` will refuse to submit or behave unexpectedly without sweeping first.

**How to apply:** Any time a payu job has failed and needs to be resubmitted, the pattern is:
```bash
payu sweep
payu run
```
