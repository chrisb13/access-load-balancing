# Memory Index

- [SSH heredoc variable expansion bug](feedback_ssh_heredoc.md) — use Python to write multi-line files to Gadi via SSH, not heredoc inside double-quoted SSH strings
- [Gadi Python/Jupyter environment](reference_gadi_python_env.md) — load `conda/analysis3` from `/g/data/xp65/public/modules`; nbconvert cwd note
- [Bash loop style](feedback_bash_loops.md) — always echo progress in loops; avoid octal-interpreted array keys like [08]/[09]
- [Payu sweep before run](feedback_payu_sweep.md) — after a failed payu job, always `payu sweep` before `payu run`
- [ACCESS-OM3 short run restart config](feedback_access_om3_short_runs.md) — set `restart_n=35, restart_option=ndays` in nuopc.runconfig to match stop length; `restart_freq` in config.yaml alone is not sufficient
- [Gadi qstat path](reference_gadi_qstat.md) — qstat not in default SSH PATH; use `/opt/nci/pbs/2024.1.2.20241017100211/bin/qstat -u cyb561`
- [Gadi payu module](reference_gadi_payu.md) — load payu via `module purge; module use /g/data/vk83/modules; module load payu`
- [Bash script invocation with flags](feedback_bash_script_invocation.md) — never call `bash script.sh -flag`; bash interprets the flag as its own; call the script directly instead
- [Gadi normalsr walltime cap](feedback_gadi_normalsr_walltime.md) — normalsr queue hard-limits walltime to 5h at large core counts (≥4992 CPUs); never request more than 05:00:00
