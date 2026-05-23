---
description: Set up and execute a cobaya MCMC run end-to-end (write YAML, install packages, launch chain). Two styles — cosmo (default; samples cosmological params with CAMB/CLASS) or halo-fit (fixed cosmology + sampled halo-model/astro params via class_sz no-cosmo mode). Forks into the cobaya-engineer subagent so install/chain output doesn't flood the main thread.
disable-model-invocation: true
argument-hint: "<run-name> [--style=cosmo|halo-fit] [likelihood-set-or-data-dir]"
context: fork
agent: cobaya-engineer
allowed-tools: Read Write Edit Glob Grep Bash(cobaya-install *) Bash(cobaya-run *) Bash(mpirun *)
---

# Set up a cobaya run

Set up and execute a cobaya MCMC run named `$0`. Pick the style by what's being fit.

Arguments in `$ARGUMENTS`:
- `$0` — run name (becomes the YAML stem and the `output:` prefix)
- `--style=cosmo` (default) or `--style=halo-fit`
- Trailing positional:
  - in `cosmo` style: comma-separated likelihood set (default `planck_2018_lowl.TT,planck_2018_lowl.EE`)
  - in `halo-fit` style: workdir path containing `data/` (the bandpower / cov / multipoles files)

## Style: `cosmo` (default)

Sample cosmological parameters against one or more CMB / LSS likelihoods.

1. **Decide theory + likelihoods.** Default theory: `camb`. Likelihoods come from the trailing positional or the default above.
2. **Write `<run-name>.yaml`** with:
   - the chosen `likelihood` block
   - `theory: { camb: { stop_at_error: True } }`
   - the standard ΛCDM `params` block (logA → A_s, n_s, H0 or theta_MC_100, omega_b, omega_cdm, tau_reio, m_ncdm fixed at 0.06)
   - `sampler: { mcmc: { Rminus1_stop: 0.05, max_tries: 1000, covmat: auto, learn_proposal: True } }` for a quick test
   - `output: chains/<run-name>`
   - `debug: True`, `packages_path: packages/`
3. **Install packages** if missing: `cobaya-install -y -p packages/ <run-name>.yaml`.
4. **Launch**: `cobaya-run <run-name>.yaml`. For MPI: `mpirun -np N cobaya-run …`.

## Style: `halo-fit`

Fit halo-model / astro parameters (pressure profile, HOD, mass bias) against a binned dataset, with cosmology **fixed**. Used for tSZ bandpowers, kSZ², galaxy×lensing astro fits, etc.

This style assumes the class_sz cobaya wrapper. For the full y-map flow with a standalone Likelihood, the **`/class-sz:build-likelihood`** skill is the right entry point — invoke it instead if the class-sz plugin is loaded.

1. **Workdir convention.** Expect a workdir layout:
   ```
   <workdir>/
   ├── <likelihood-module>.py       # likelihood class to fit
   ├── data/                        # bandpowers, cov, multipoles, foreground template
   └── chains/                      # output (initially empty)
   ```
   `cd <workdir>` before running so the likelihood module is importable.

2. **Write `<run-name>.yaml`** with:
   - `theory: { classy_szfast.classy_sz.classy_sz: { use_class_sz_fast_mode: 1, use_class_sz_no_cosmo_mode: 1, extra_args: { ... cosmology fixed here ... } } }`
   - `likelihood: { <module>.<Class>: { sz_data_directory: <workdir>/data/, ymap_ps_file: …, ymap_cov_file: … } }`
   - sampled `params` block — typically just the astro/profile params (e.g. `P0GNFW`, `betaGNFW`); foregrounds fixed at 0
   - `sampler: { mcmc: { Rminus1_stop: 0.05, max_tries: 10000, learn_proposal: True, blocking: [[1, [P0GNFW, betaGNFW]]] } }`
   - `output: <workdir>/chains/<run-name>`
   - `debug: True`, `timing: true`
3. **No `cobaya-install`** needed — class_sz / classy_szfast are imported directly from the venv.
4. **Validate** with `cobaya-run --test <run-name>.yaml` first.
5. **Launch**: `cobaya-run <run-name>.yaml` (or with mpirun).

## Common: report back

- Style used, files written (absolute paths)
- `--test` result if used
- Chain location, max `|R-1|` reached, total samples
- Acceptance rate from `.progress`
- Any theory crashes (grep `chains/<run-name>.log` or stderr for `error`/`Traceback`)
- Next-step suggestion (tighten `Rminus1_stop`, add more likelihoods, switch to MPI, etc.)

## Constraints
- Do not `pip install` cobaya or its deps — assume the venv is complete.
- Do not silently swallow theory errors; report them.
- Do not overwrite an existing chain — use `resume: True` or pick a new name.
- Quote `${VARS}` in any shell command you generate.
- Do not run from a directory containing a `cobaya/` subdir (PEP 420 cwd collision; see `/cobaya:explain` Pitfalls).
