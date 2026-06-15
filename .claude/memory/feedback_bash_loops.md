---
name: feedback-bash-loops
description: Always include an echo statement in bash loops to show progress; avoid octal-interpreted indices
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

Always include an `echo` in bash loops to indicate where we are, e.g. `echo "Submitting test${T}..."`.

**Why:** User expects progress feedback when running multi-step loops.

**How to apply:** Any time I write a bash for-loop that runs multiple commands, add an echo at the top of the loop body.

Also: avoid associative arrays with keys like `[08]`/`[09]` — bash interprets these as octal and errors. Use parallel indexed arrays instead.
