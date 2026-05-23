---
description: Cobaya cosmological Bayesian framework — likelihoods, theories, samplers, MCMC/PolyChord, Planck/CLASS/CAMB integration. Use when writing or debugging cobaya YAML, Likelihood subclasses, sampler config, chain analysis with getdist, or cobaya-run/cobaya-install invocations.
when_to_use: User mentions cobaya, MCMC convergence, Rminus1, Planck likelihoods, classy/camb theory, getdist, chain plotting, derived parameters, external priors, or cobaya-install.
allowed-tools: Read Grep Glob Bash(cobaya-run *) Bash(cobaya-install *) Bash(getdist *)
---

# Cobaya assistant

Help the user with the Cobaya Bayesian-analysis framework (docs: https://cobaya.readthedocs.io · source: https://github.com/CobayaSampler/cobaya).

## Anatomy of a run
Every cobaya run has four blocks:
- **`params`** — fixed / sampled (with `prior`) / derived (lambda or theory output)
- **`theory`** — `camb`, `classy`, or a `cobaya.theory.Theory` subclass
- **`likelihood`** — built-in (`planck_2018_lowl.TT`, `sn.pantheon`) or `Likelihood` subclasses
- **`sampler`** — usually `mcmc` (Metropolis with covmat learning), `polychord` for nested, `evaluate`/`minimize` otherwise

Outputs land at `output: chains/<name>` as `<name>.{1,2,...}.txt` plus `.input.yaml`, `.updated.yaml`, `.progress`.

## Two entry points (equivalent)

**YAML**, run with `cobaya-run my.yaml`:
```yaml
likelihood:
  gaussian_mixture: { means: [0.2, 0], covs: [[0.1, 0.05], [0.05, 0.2]] }
params:
  a: { prior: {min: -0.5, max: 3}, ref: 0.5, proposal: 0.3 }
  b: { prior: {dist: norm, loc: 0, scale: 1}, ref: 0, proposal: 0.5 }
sampler:
  mcmc: { Rminus1_stop: 0.01, max_tries: 1000, covmat: auto }
output: chains/run1
```

**Python**:
```python
from cobaya import run
updated_info, sampler = run(info)
```
For non-sampling evaluation: `cobaya.model.get_model(info).logposterior(params)`.

## Custom likelihood skeleton
```python
from cobaya.likelihood import Likelihood

class MyLike(Likelihood):
    sigma: float = 0.1                   # class attrs are YAML-overridable
    def initialize(self): ...            # called once
    def get_requirements(self):
        return {"Cl": {"tt": 2500}}
    def logp(self, **params_values):
        cl = self.provider.get_Cl(ell_factor=True)
        return float(...)
```
Wire it in:
```yaml
likelihood:
  my_like:
    external: import_module('my_module').MyLike
    sigma: 0.05
    output_params: [Map_Cl_at_500]    # declare any derived it computes
```

## Custom theory skeleton
```python
from cobaya.theory import Theory

class MyTheory(Theory):
    def get_requirements(self): return {}
    def calculate(self, state, want_derived=True, **params_values):
        state["my_quantity"] = compute(...)
    def get_can_provide(self): return ["my_quantity"]
```

## Standard ΛCDM parameter block (CAMB or CLASS)
```yaml
params:
  logA:    { prior: {min: 2, max: 4}, ref: {dist: norm, loc: 3.05, scale: 0.001}, proposal: 0.001, drop: true, latex: '\log(10^{10} A_s)' }
  A_s:     { value: 'lambda logA: 1e-10*np.exp(logA)', latex: 'A_s' }
  n_s:     { prior: {min: 0.8, max: 1.2}, ref: {dist: norm, loc: 0.96, scale: 0.004}, proposal: 0.002 }
  H0:      { prior: {min: 40, max: 100}, ref: {dist: norm, loc: 70, scale: 2}, proposal: 2 }
  omega_b: { prior: {min: 0.005, max: 0.1}, ref: {dist: norm, loc: 0.0221, scale: 0.0001}, proposal: 0.0001 }
  omega_cdm: { prior: {min: 0.001, max: 0.99}, ref: {dist: norm, loc: 0.12, scale: 0.001}, proposal: 0.0005 }
  m_ncdm:  { renames: mnu, value: 0.06 }
  tau_reio: { prior: {min: 0.01, max: 0.8}, ref: {dist: norm, loc: 0.06, scale: 0.01}, proposal: 0.005 }
```

## Debug order for a stuck chain
1. `debug: True` in YAML
2. `theory: {camb: {stop_at_error: True}}` to surface theory crashes
3. Loosen `Rminus1_stop` for quick tests (0.05 instead of 0.01)
4. `covmat: auto` to bootstrap from cobaya's cosmology covmat database
5. Lower `proposal` for tight priors, raise `max_tries`
6. Check `.progress` files for acceptance rate (~0.2–0.3 is healthy)

## Chain analysis with getdist
```python
from getdist import loadMCSamples
s = loadMCSamples("chains/run1", settings={"ignore_rows": 0.3})
print(s.getTable(limit=1).tableTex())
```

## Pitfalls
- `output: null` keeps chains in memory only — they vanish at exit
- `drop: true` prevents a param from being passed to the theory (used for `logA` when deriving `A_s`)
- Don't set both `value` and `prior` on the same param
- `stop_at_error: false` (theory default) silently returns `-inf` on theory failure
- `renames: mnu` lets one parameter answer to multiple names (cosmology mappings between CAMB and CLASS)
- For CMB use `theta_MC_100` or `theta` rather than sampling `H0` directly when comparing to Planck
- **Cwd-collision footgun for editable cobaya installs.** If you have a local clone of cobaya at `<dir>/cobaya/` and run python from `<dir>`, Python's PEP 420 namespace-package machinery treats `cobaya` as an empty namespace package shadowing the editable install. Symptoms: `cobaya.__file__` is `None`, `dir(cobaya)` is empty, `from cobaya import LoggedError` fails, and downstream packages (e.g. soliket) blow up on import. Fix: `cd` somewhere without a `cobaya/` subdir (`/tmp`, your workdir, etc.) before invoking python or cobaya-run.

## Recipe — rebase a reference YAML to a local workdir

Common scenario: a known-good YAML from another machine has absolute paths that don't exist locally. To make it runnable in a self-contained workdir:

1. Copy the YAML into `<workdir>/reference/<name>.yaml` as a read-only baseline.
2. Make a runnable copy at `<workdir>/reference/<name>.local.yaml`.
3. Rewrite paths in the copy:
   - data dirs → `<workdir>/data/`
   - external module refs (likelihood, theory) → either install the module or vendor a standalone replacement at `<workdir>/<module>.py` and update the dotted ref
   - sampler `covmat:` → drop it, or point at a covmat copied into `<workdir>/reference/`
   - `output:` → `<workdir>/chains/<name>.local`
4. `cd <workdir>` before running so vendored modules are importable.
5. `cobaya-run --test <…>.local.yaml` first — catches all path/import errors in seconds; then run for real.

## Going deeper
For full YAML schema, all sampler options, PolyChord nested sampling, Planck likelihood module names, and `Provider` API see [reference.md](reference.md).
