# Cobaya Assistant (Claude Code plugin)

Specialized [Claude Code](https://code.claude.com) assistance for the [Cobaya](https://cobaya.readthedocs.io) cosmological Bayesian-analysis framework.

## What you get

- **`/cobaya:explain`** — passive knowledge skill: cobaya conventions, likelihood/theory skeletons, sampler config, debug order. Auto-loads when the conversation is about cobaya, MCMC convergence, Planck likelihoods, getdist, etc.
- **`/cobaya:setup-run <name> [likelihoods]`** — sets up and executes a full cobaya MCMC run end-to-end. Forks into the `cobaya-engineer` subagent so install/chain logs don't flood the main thread; returns a concise summary.
- **`cobaya-engineer` subagent** — specialist subagent for cobaya runs; can be invoked directly from `/agents` or used as the executor for `/cobaya:setup-run`.

## Install

In any Claude Code session, run these three commands:

```
/plugin marketplace add https://github.com/borisbolliet/cobaya-claude-plugin.git
/plugin install cobaya@cobaya-claude-plugin
/reload-plugins
```

After `/reload-plugins`, `/cobaya:explain` and `/cobaya:setup-run`
show in `/help` and the `cobaya-engineer` subagent appears in the
Agent picker.

To update later (after I push a new commit):

```
/plugin uninstall cobaya@cobaya-claude-plugin
/plugin marketplace remove cobaya-claude-plugin
/plugin marketplace add https://github.com/borisbolliet/cobaya-claude-plugin.git
/plugin install cobaya@cobaya-claude-plugin
/reload-plugins
```

## Try it

```
/cobaya:explain
What's the right covmat strategy for a Planck+lensing chain?

/cobaya:setup-run lcdm_planck planck_2018_lowl.TT,planck_2018_lowl.EE
```

## Layout

```
.claude-plugin/marketplace.json   # single-plugin marketplace
plugins/cobaya/
  .claude-plugin/plugin.json      # plugin manifest
  skills/
    explain/
      SKILL.md                    # main knowledge skill
      reference.md                # detailed schema (loaded on demand)
    setup-run/
      SKILL.md                    # forks into cobaya-engineer
  agents/
    cobaya-engineer.md            # the subagent
```

## Notes / open items

- **No MCP server.** Cobaya's CLI (`cobaya-run`, `cobaya-install`, `getdist`) is invoked directly via the Bash tool. An MCP server only becomes worthwhile if you want a typed tool surface or a long-lived/remote service (warm CAMB process, cluster job submitter).
- **Agent reference in `context: fork`.** `skills/setup-run/SKILL.md` references the subagent as `agent: cobaya-engineer`. If the plugin namespace isn't picked up automatically, switch to `agent: cobaya:cobaya-engineer` or whatever the live `/agents` listing shows.
- **`allowed-tools`** on the skills pre-approves `cobaya-run`, `cobaya-install`, `getdist`. Bash invocations that don't match those patterns will still prompt.
- **No `paths` filter** on `/cobaya:explain` — it's available everywhere, auto-loaded by description rather than file context.

## License

MIT
