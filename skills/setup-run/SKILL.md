---
description: Set up and execute a cobaya MCMC run end-to-end (write YAML, install packages, launch chain). Forks into the cobaya-engineer subagent so install/chain output doesn't flood the main thread.
disable-model-invocation: true
argument-hint: <run-name> [likelihood-set]
context: fork
agent: cobaya-engineer
allowed-tools: Read Write Edit Glob Grep Bash(cobaya-install *) Bash(cobaya-run *) Bash(mpirun *)
---

# Set up a cobaya run

Set up and execute a cobaya MCMC run named `$0`. Optional second argument: a comma-separated likelihood set (default: `planck_2018_lowl.TT,planck_2018_lowl.EE`).

## Steps

1. **Decide theory + likelihoods.** Default theory: `camb`. Default likelihoods come from `$1` or the default above. Read any existing `chains/$0.yaml` first — if it already exists, ask before overwriting.

2. **Write `chains/$0.yaml`** with:
   - the chosen `likelihood` block
   - `theory: { camb: { stop_at_error: True } }`
   - the standard ΛCDM `params` block (logA → A_s, n_s, H0 or theta_MC_100, omega_b, omega_cdm, tau_reio, m_ncdm fixed at 0.06)
   - `sampler: { mcmc: { Rminus1_stop: 0.05, max_tries: 1000, covmat: auto, learn_proposal: True } }` for a quick test
   - `output: chains/$0`
   - `debug: True`
   - `packages_path: packages/` (or `${COBAYA_PACKAGES_PATH}` if set)

3. **Install packages** if missing: `cobaya-install -y -p packages/ chains/$0.yaml`. Skip if `packages/code` and `packages/data` already exist for the requested likelihoods.

4. **Launch**: `cobaya-run chains/$0.yaml`. If MPI is available and the user asked for parallel chains, use `mpirun -np N cobaya-run …`.

5. **Report back** (concise):
   - chain location and number of chains written
   - max `|R-1|` reached and total samples
   - acceptance rate from `.progress`
   - any theory crashes (grep `chains/$0.log` or stderr for `error`/`Traceback`)
   - next step suggestion (tighten `Rminus1_stop`, add more likelihoods, etc.)

## Constraints
- Do not `pip install` cobaya or its deps — assume it's already importable.
- Do not silently swallow theory errors; report them.
- Do not overwrite an existing chain — use `resume: True` or pick a new name.
- Quote `${VARS}` in any shell command you generate.
