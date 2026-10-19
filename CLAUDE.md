# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A Salt formula managing `/etc/resolv.conf`, with or without the `resolvconf`
package. It is the outlier of the `salt-formulas/` collection (see the parent
`CLAUDE.md`): it predates `template-formula`, so none of the collection-wide
conventions about `docs/`, `test/`, `_mapdata`, TOFS (`libtofs.jinja`) or the
three-platform `kitchen.yml` apply here.

- No `kitchen.yml`, only a legacy `.kitchen.yml`; no `.github/` workflow, so
  pushes run no CI.
- No `test/` tree: the single Kitchen suite only converges, nothing is verified.
- `FORMULA` has no `top_level_dir` and no `minimum_version`; `version: 201801`
  is stale.
- README is `README.rst` at the root; `pillar.example` documents both variants.

## Commands

```bash
bundle install
bin/install-hooks                               # pre-commit + commit-msg hooks
pre-commit run --all-files                      # yamllint, salt-lint, rubocop, shellcheck, rstcheck, rst-lint

bin/kitchen list                                # one instance: resolver-ng-debian-11-master-py3
bin/kitchen converge resolver-ng-debian-11-master-py3
bin/kitchen destroy
```

The only Kitchen platform is `saltimages/salt-master-py3:debian-11` (Salt
master branch, not a released version), and the suite applies `resolver.ng` with
`pillar.example` as pillar — it never exercises the classic `resolver` state.
`bin/kitchen list` writes `.kitchen/` logs; there is no `.gitignore`, so remove
that directory rather than let it show up as untracked.

Here `.yamllint` caps lines at 100 (the collection-wide 88 does not apply);
`.rubocop.yml` caps Ruby at 88. salt-lint skips 205, 207, 208.

## Architecture: two independent states in one tree

`resolver/` holds two unrelated implementations that share only the pillar
root key `resolver` and the `resolvconf`-then-`resolv.conf` flow.

### `resolver` (classic) — `resolver/init.sls`

- `map.jinja` is an inline dict passed to `grains.filter_by` on the `os` grain
  (not `os_family`), merged with the whole `resolver` pillar. It only supplies
  `conf_path`, `user`, `group`.
- The switch is `pillar resolver:use_resolvconf` (default `True`):
  `resolvconf` is kept only if that is true **and** `os_family == Debian`;
  otherwise the package is purged and `/etc/resolv.conf` is written directly.
  With resolvconf, the file written is `/etc/resolvconf/resolv.conf.d/base`,
  `/etc/resolv.conf` is symlinked to `conf_path`, and `resolvconf -u` runs on
  change.
- `nameservers`, `searchpaths`, `options`, `domain` are read straight from the
  pillar in `init.sls` (not from the map), with defaults there:
  `8.8.8.8/8.8.4.4`, the `domain` grain, `[]`. `domain` has no default, so an
  unset key renders `domain None` in `files/resolv.conf`.
- The template source is hardcoded as `salt://resolver/files/resolv.conf`; a
  deployment overrides it by shadowing that path on the fileserver.

### `resolver.ng` — `resolver/ng/`

- `ng/map.jinja` layers `defaults.yaml` (under a `Defaults` key) ←
  `osfamilymap.yaml` ← `osmap.yaml` ← `osfingermap.yaml` with
  `defaults.merge`, injects the `domain` grain, then merges the `resolver`
  pillar on top. All ng-specific settings live under `resolver:ng:*`; the
  record values (`nameservers`, …) stay at `resolver:*`.
- `ng.resolvconf.enabled` / `.remove` decide install vs purge and which file is
  written; Debian enables resolvconf by default (`osfamilymap.yaml`), and the
  symlink target per release is set in `osfingermap.yaml`.
- Unlike the classic state, it also sets `dns=none` in NetworkManager's config
  (`ini.options_present`) and restarts the service, when the config file
  exists, `ng.networkmanager.managed` is true and the unit is enabled.
- State IDs are `{{ sls }}~…` and the template path uses `slspath`, so the tree
  must be served as `resolver/`.

Both templates (`files/resolv.conf`, `ng/templates/resolv.conf.jinja`) render
the same directives in the same order: `domain`, `search`, `nameserver`, sorted
`options`.
