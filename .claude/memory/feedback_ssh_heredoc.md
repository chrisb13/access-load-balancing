---
name: feedback-ssh-heredoc
description: "When writing files to Gadi via SSH with heredoc, variables inside the heredoc get expanded by the local shell if inside a double-quoted SSH string — use Python to write multi-line files instead"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

Never use `ssh gadi "cat > file << 'EOF'\n...\nEOF"` to write files containing shell variables like `${VAR}`. Even with `'EOF'` (single-quoted), the outer local shell expands `${VAR}` to empty before sending to SSH.

**Why:** The double-quoted SSH string causes local shell expansion of `${}` before the remote shell ever sees the heredoc. The `'EOF'` only prevents expansion on the remote shell, which is too late.

**How to apply:** Use Python to write multi-line files to Gadi:
```bash
ssh gadi "python3 << 'PYEOF'
content = '''...file content with ${VAR} as literals...'''
open('/path/to/file', 'w').write(content)
PYEOF"
```
The Python triple-quoted string doesn't interpret `${}`, so variables are written literally and expand correctly when the PBS job runs.
