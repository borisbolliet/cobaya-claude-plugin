---
description: Set up and execute a cobaya MCMC run end-to-end (write YAML, install packages, launch chain). Forks into the cobaya-engineer subagent so install/chain output doesn't flood the main thread.
disable-model-invocation: true
argument-hint: "<run-name> [hints...]"
context: fork
agent: cobaya-engineer
allowed-tools: Read Write Edit Glob Grep Bash(cobaya-install *) Bash(cobaya-run *) Bash(mpirun *)
---

# Set up a cobaya run

Set up and execute a cobaya MCMC run named `$0`. The skill assembles a YAML from four independent decisions; treat them separately and ask the user when underspecified — don't pre-bin into preset "styles".

The four decisions:

1. **Likelihood(s)** — Planck / ACT / DES / a tSZ Cl^yy bandpower file / joint / etc.
2. **Theory code** — `camb` (CAMB), `classy` (CLASS), or `classy_szfast.classy_sz.classy_sz` (class_sz; itself a full Boltzmann code with CosmoPower emulators in fast mode, plus halo-model observables). Pick what supports the requested observables.
3. **What's sampled vs fixed** — any subset of cosmology, astro/HOD/profile, nuisances, foregrounds. Cosmology can be fully sampled, fully fixed in `extra_args` (e.g. with `use_class_sz_no_cosmo_mode: 1` for class_sz), or partially constrained. Same for astro/nuisances.
4. **Output** — chain location, debug, timing, MPI, `Rminus1_stop` target, starting `covmat`.

Hints in `$ARGUMENTS` after `$0` can shortcut any of these (e.g. `planck_2018_lowl.TT,planck_2018_lowl.EE` → likelihood set; `--theory=class_sz` → theory choice; `--data-dir=…` → bandpower workdir). Ask for anything left underspecified before writing.

## Steps

1. **Resolve the four decisions.** From `$ARGUMENTS`, existing files in the workdir (e.g. an existing `<run-name>.yaml` to amend, a `data/` dir, a `reference-*/` baseline), and the user.

2. **Sketch the YAML** with the four standard blocks:
   - `theory: { <code>: { stop_at_error: True, [extra_args: { ... }] } }`
   - `likelihood: { <name1>: null, <name2>: { ... } }`
   - `params: { ... fixed values, sampled priors, derived ... }`
   - `sampler: { mcmc: { Rminus1_stop: 0.05, max_tries: 1000, learn_proposal: True, [covmat: ...] } }`
   - `output: chains/<run-name>` (or absolute path inside a workdir)
   - `debug: True`, `timing: true`

3. **Install external packages** if any are referenced (e.g. Planck likelihoods):
   `cobaya-install -y -p packages/ <run-name>.yaml`. Skip for runs that only use in-venv code (class_sz, classy_szfast, a standalone Likelihood module).

4. **Validate with `--test`** before launching:
   `cobaya-run --test <run-name>.yaml`. Reports the model assembly, initial point, per-component speed measurements; takes seconds.

5. **Launch**:
   - Single chain: `cobaya-run <run-name>.yaml`
   - MPI: `mpirun -np N cobaya-run <run-name>.yaml` (N parallel chains; bump for more cores)

6. **Report back**:
   - YAML path written, `--test` result
   - Chain location, max `|R-1|` reached, total samples, n_chains, acceptance rate from `.progress`
   - Any theory crashes (`Traceback`/`error` in stderr or `<run-name>.log`)
   - One-line suggested next step (tighten `Rminus1_stop`, switch to MPI, add a likelihood, etc.)

## Cross-plugin handoffs

- For a **tSZ Cl^yy power-spectrum bandpower likelihood** (binned bandpowers + N×N covariance, fit via class_sz), the class-sz plugin's **`/class-sz:build-likelihood`** is the right entry point if loaded — it scaffolds a standalone likelihood module + a working YAML in one shot.
- For deeper class_sz / classy_szfast questions, defer to `/class-sz:explain`.

## Workdir convention (when external module/data are involved)

```
<workdir>/
├── <likelihood-module>.py         # standalone Likelihood/Theory (optional)
├── <run-name>.yaml
├── data/                          # whatever the likelihood reads
└── chains/                        # cobaya output
```
`cd <workdir>` before running cobaya-run so any local module is importable.

## Constraints

- Do not `pip install` cobaya or its deps — assume the venv is complete.
- Do not silently swallow theory errors; report them.
- Do not overwrite an existing chain — use `resume: True` or pick a new name.
- Quote `${VARS}` and any path with spaces.
- Don't run from a directory that has a `cobaya/` subdir (PEP 420 cwd collision — see `/cobaya:explain` Pitfalls).
