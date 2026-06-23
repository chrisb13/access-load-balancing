---
name: feedback-ssh-heredoc
description: "When writing files to Gadi via SSH with heredoc, variables inside the heredoc get expanded by the local shell if inside a double-quoted SSH string — use Python with literal paths to write multi-line files instead"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

Never use `ssh gadi "cat > file << 'EOF'\n...\nEOF"` to write files containing shell variables like `${VAR}`. Even with `'EOF'` (single-quoted), the outer local shell expands `${VAR}` to empty before sending to SSH.

**Why:** The double-quoted SSH string causes LOCAL shell expansion of `${}` before the remote shell ever sees the heredoc. The `'EOF'` only prevents expansion on the REMOTE shell, which is too late. This applies equally to content inside a Python `'''triple-quoted string'''` passed via the heredoc — the local bash expands `${}` before Python ever sees the content. This burned us twice: once with `${PBS_NCI_STORAGE}` type variables, and again with `${SCRIPT}`, `${HGRID}`, `${TOPOG}` in a PBS job body, causing those variables to expand to empty and leaving a bare `-g` as a command.

**How to apply:** Use Python to write multi-line files to Gadi, AND avoid `${VAR}` patterns in the content — use literal values instead:
```bash
ssh gadi "python3 << 'PYEOF'
content = '''/path/to/tool -g /literal/path/hgrid.nc -t /literal/path/topog.nc -r 3500 4800
'''
open('/path/to/file', 'w').write(content)
PYEOF"
```

If you genuinely need `${VAR}` to appear literally in the written file (e.g. a PBS script that uses those variables at runtime), escape the dollar sign so the local bash does not expand it: write `\${VAR}` inside the Python content string — the local bash converts `\$` to `$`, so the file receives the literal `${VAR}` text.

If the Python code itself needs literal double-quote characters (e.g. regex patterns matching `"..."` strings), do NOT write a separate script file — use `chr(34)` instead of `"` in the Python source inside the PYEOF heredoc:
```python
q = chr(34)
txt = re.sub('MASKTABLE = ' + q + '[^' + q + ']+' + q, 'MASKTABLE = ' + q + new_val + q, txt)
```
This avoids the double-quote terminating the outer SSH double-quoted string, and there is no need for a write-to-file-then-execute two-step.
