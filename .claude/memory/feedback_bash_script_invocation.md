---
name: feedback-bash-script-invocation
description: Never call a shell script with "bash script.sh -arg" — bash interprets the flags, not the script; call the script directly instead
metadata:
  type: feedback
---

Never prefix a shell script with `bash` when passing flags to the script itself.

**Why:** `bash script.sh -g /path/to/file` makes bash interpret `-g` as a bash option (invalid), not as an argument to the script. The script never runs. This bit us when calling `gen_masktable.sh -g HGRID -t TOPOG -r MIN MAX` — bash exited with "invalid option" immediately.

**How to apply:** Call scripts directly (they have their own shebang) or use `sh` only when you want POSIX shell, not to pass script arguments:

```bash
# Wrong — bash sees -g as its own flag:
bash /path/to/gen_masktable.sh -g hgrid.nc -t topog.nc -r 3500 4800

# Correct — script's shebang handles the interpreter:
/path/to/gen_masktable.sh -g hgrid.nc -t topog.nc -r 3500 4800

# Also correct if script is not executable:
bash /path/to/gen_masktable.sh   # no flags to the script itself
```
