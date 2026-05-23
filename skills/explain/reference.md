# Cobaya — detailed reference

Load this when a question needs schema-level detail beyond what SKILL.md covers. Source: https://cobaya.readthedocs.io

## Top-level YAML blocks
| Block | Type | Notes |
| --- | --- | --- |
| `params` | dict | name → param spec |
| `prior` | dict | name → string defining a multidim log-prior (e.g. `"lambda a,b: stats.norm.logpdf(a-b, 0, 0.5)"`) |
| `likelihood` | dict | name → likelihood spec |
| `theory` | dict | name → theory spec |
| `sampler` | dict | exactly one of: `mcmc`, `polychord`, `evaluate`, `minimize` |
| `output` | str or null | base path for chains and metadata; `null` = in-memory only |
| `debug` | bool/str | True for verbose logs; can be a log path |
| `resume` | bool | continue chains at `output` if they exist |
| `force` | bool | overwrite chains at `output` |
| `stop_at_error` | bool | global default for components |
| `packages_path` | str | where `cobaya-install` puts external packages |
| `test` | bool | dry-run model construction without sampling |
| `timing` | bool | report per-component timings on exit |

## Parameter spec
```yaml
my_param:
  prior:                       # one of:
    min: 0; max: 1             #   uniform
    dist: norm; loc: 0; scale: 1  # any scipy.stats distribution
  ref:                         # starting point (scalar or distribution)
  proposal: 0.1                # initial proposal std (mcmc only)
  latex: '\alpha'
  drop: true                   # don't pass to theory/likelihood
  renames: [name1, name2]      # accept alternative names
  derived: "lambda a, b: np.sqrt(a**2 + b**2)"   # derived from sampled
  value: 0.06                  # fixed; OR "lambda ...:" for derived from sampled
  min: 0; max: 1               # hard bounds beyond the prior (for derived clipping)
```
A param is **sampled** if it has `prior`, **fixed** if it has `value` (scalar), **derived** if it has `derived` (lambda) or appears only as a theory/likelihood output.

## Sampler — `mcmc`
| Field | Default | Purpose |
| --- | --- | --- |
| `Rminus1_stop` | 0.01 | convergence threshold on max(|R-1|) |
| `Rminus1_cl_stop` | 0.2 | extra check on 95% CL bounds |
| `max_tries` | 40d | max proposal tries per accepted point (d = dim) |
| `max_samples` | inf | hard stop |
| `burn_in` | 0 | initial samples discarded |
| `learn_proposal` | true | adapt proposal covariance |
| `learn_proposal_Rminus1_max` | 2.0 | only learn when R-1 < this |
| `covmat` | none | starting covariance: `auto` searches built-in DB, or a path |
| `proposal_scale` | 2.4 | overall scale factor |
| `drag` | false | dragging for fast/slow split (cosmology) |
| `oversample_power` | 0.4 | oversampling for fast params |
| `blocking` | auto | manual parameter blocking |
| `output_every` | 60s | flush cadence |

## Sampler — `polychord`
| Field | Default | Purpose |
| --- | --- | --- |
| `nlive` | 25d | live points |
| `precision_criterion` | 0.001 | stopping criterion |
| `num_repeats` | 2*d | repeats per slice step |
| `do_clustering` | true | cluster recognition for multimodal posteriors |
| `nprior` | 10*nlive | initial samples from prior |
| `feedback` | 1 | verbosity 0-3 |

## Likelihood base API
```python
class MyLike(Likelihood):
    # class attrs become YAML options
    speed: int = 1           # for fast/slow blocking
    type: str = "CMB"        # group tag

    def initialize(self): ...
    def get_requirements(self):
        # ask theory for quantities
        return {"Cl": {"tt": 2500}, "H": {"z": [0.0, 1.0]}}
    def get_can_provide_params(self):
        return ["derived_name"]   # parameters this likelihood derives
    def logp(self, **params_values):
        cl = self.provider.get_Cl(ell_factor=True)
        # ... return float log-likelihood
    def close(self): ...     # cleanup
```

## Theory base API
```python
class MyTheory(Theory):
    speed: int = -1
    def initialize(self): ...
    def get_requirements(self):
        return {}                          # what this theory needs from other components
    def must_provide(self, **requirements):
        # called once per session with all aggregated requirements
        return {}                          # what THIS needs from upstream
    def calculate(self, state, want_derived=True, **params):
        state["my_qty"] = ...              # compute and store
        if want_derived: state["derived"] = {"foo": ...}
    def get_can_provide(self):
        return ["my_qty"]
    def get_can_provide_params(self):
        return ["foo"]
    def get_my_qty(self): return self.current_state["my_qty"]   # accessor for likelihoods
```

## `Provider` accessors (called from likelihoods)
| Method | Returns |
| --- | --- |
| `get_param(name)` | sampled or derived value |
| `get_Cl(ell_factor=True, units="muK2")` | CMB power spectra dict |
| `get_Hubble(z)` | H(z) array |
| `get_angular_diameter_distance(z)` | D_A(z) |
| `get_comoving_radial_distance(z)` | D_C(z) |
| `get_sigma8_z(z)` | σ8(z) |
| `get_fsigma8(z)` | fσ8(z) |
| `get_Pk_interpolator(...)` | matter power spectrum interpolator |
| `get_<custom>()` | from a custom Theory's `get_can_provide` |

## Built-in Planck likelihood modules (representative)
- `planck_2018_lowl.TT` / `.TT_native` / `.EE` / `.EE_sroll2`
- `planck_2018_highl_plik.{TT,TTTEEE,TT_lite,TTTEEE_lite,TT_unbinned}`
- `planck_2018_highl_CamSpec.{TT,TTTEEE,TT_native,TTTEEE_native}`
- `planck_NPIPE_highl_CamSpec.{TT,TTTEEE}`
- `planck_2018_lensing.{clik,native}`

Install with `cobaya-install -p packages/ <yaml>`.

## External (function) likelihood
```yaml
likelihood:
  my_like:
    external: "lambda a, b: -0.5*((a-1)**2 + b**2)/0.01"
    input_params: [a, b]
```
Or a Python callable referenced as `external: import_module('mymod').my_func`.

## External prior
Top-level `prior:` block. Each value is a string defining a log-prior:
```yaml
prior:
  band_constraint: "lambda a, b: stats.norm.logpdf(a-b, loc=0, scale=0.5)"
```
The function must depend only on sampled parameter names.

## `output` layout
```
chains/run1.input.yaml      # exact input as parsed
chains/run1.updated.yaml    # input + defaults + derived requirements
chains/run1.1.txt           # chain 1 (weight, -logL, params, derived)
chains/run1.2.txt           # chain 2 (parallel MPI rank)
chains/run1.progress        # acceptance rate / R-1 history
chains/run1.locked          # session lock (mcmc)
chains/run1.checkpoint      # for resume
```

## Useful CLI
```bash
cobaya-run input.yaml                          # run
cobaya-run input.yaml --resume                 # continue existing chain
cobaya-run input.yaml --force                  # overwrite
cobaya-run input.yaml --test                   # build model, don't sample
cobaya-install -p packages/ input.yaml         # install external packages
cobaya-install -p packages/ planck_2018_lowl.TT  # install a single component
cobaya-bib input.yaml                          # print BibTeX for components used
cobaya-doc <component>                         # print docs / default options
cobaya-grid-create grid.yaml                   # multi-run grid
mpirun -np 4 cobaya-run input.yaml             # MPI: 4 parallel chains
```

## Patterns

**Sampling `theta` instead of `H0`** (faster for Planck):
```yaml
params:
  theta_MC_100: { prior: {min: 0.5, max: 10}, ref: {dist: norm, loc: 1.04109, scale: 0.0004}, proposal: 0.0002, drop: true, renames: theta }
  H0: { latex: 'H_0' }   # derived from theta + other params by CAMB
```

**Fast/slow blocking** for cosmology (CMB nuisance vs cosmological params): set `speed` on likelihoods/theories and `drag: true` on mcmc, or specify `blocking:` explicitly.

**Resuming with new likelihoods**: use `force_reproducible: false` and add the new likelihood — cobaya warns about input mismatch but proceeds.

**Multiple likelihoods sharing a derived param**: declare it once with `output_params:` on one likelihood; others reference it via `get_param`.
