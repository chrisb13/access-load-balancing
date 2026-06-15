---
name: reference-gadi-python-env
description: How to load Python/Jupyter on Gadi for scientific analysis (xp65 analysis3 conda environment)
metadata: 
  node_type: memory
  type: reference
  originSessionId: 04391d37-719d-42cf-bf0b-c84bc991b038
---

To use Python with scientific packages (numpy, pandas, matplotlib, jupyter, nbconvert) on Gadi:

```bash
module use /g/data/xp65/public/modules
module load conda/analysis3
```

Docs: http://docs.access-hive.org.au/getting_started/environments/#how-to-use-the-xp65-condaanalysis3-environment

To execute a Jupyter notebook in-place (non-interactively):
```bash
module use /g/data/xp65/public/modules && module load conda/analysis3
cd <notebook_parent_dir>
jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=300 notebook.ipynb
```

Note: nbconvert changes cwd to the notebook's own directory during execution. If the notebook uses relative paths (e.g. `HERE = 'subdir'`), run nbconvert from inside the directory the notebook was designed to run from, or adjust the path to `HERE = '.'`.
