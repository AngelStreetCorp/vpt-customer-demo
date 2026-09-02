# Demo Customer — a VirtualPyTest customer overlay

This repository is the **reference example of a customer overlay** for
[VirtualPyTest](https://github.com/AngelStreetCorp/virtualpytest). It is what runs at
`virtualpytest-demo.angelstreet.io`, and it is the template to copy when onboarding a new
customer. It contains **no secrets and no internal addresses**, so it can be public.

A customer deployment is exactly two things:

| Layer | Repo | What it holds |
|---|---|---|
| Platform | `AngelStreetCorp/virtualpytest` at a pinned release | all the code |
| Overlay | this repo (one per customer, one per environment) | branding, navigation, optional-feature choices, customer scripts and dashboards |

`deploy_customer.sh` (in the platform repo, `setup/proxmox/node/`) checks out the platform at
`PIN`, removes the disabled features, copies this overlay on top, rebuilds the frontend and
pushes the result to the customer's machines. Nothing is merged, patched or forked.

## Layout

```
customer.conf                 PIN, DISABLED_FEATURES (publishable)
customer.local.conf           TARGETS and other machine-specific overrides (git-ignored)
customer.local.conf.example   template for the file above
VERSION.txt                   this overlay's version, bumped by the pre-commit hook
frontend/.env.production      branding + navigation (VITE_* only, layered over the VM's .env)
frontend/public/brand/        logo and other brand assets
infra/monitoring/grafana/     customer dashboards (optional)
infra/proxy/nginx/config/     customer vhost confs (optional)
test_scripts/, test_campaign/ customer test scripts and identity maps (optional)
scripts/bump_version.sh, .githooks/pre-commit   same versioning as the platform
```

Only files that do **not** exist in the platform belong here, plus deliberate whole-file
overrides listed in an `OVERRIDES.md`. A stale copy of a platform file in an overlay silently
shadows the newer platform version at deploy time.

## Onboard a new customer

1. **Copy this repo** to a private repo named `vpt-customer-<codename>` (use a codename, never the
   customer's real name, in any repo, branch or folder name).
2. **Branch**: work on `prod` (the version string then reads `prod-<date>-<build>`); keep `demo` for
   demo environments.
3. **`customer.conf`**: set `PIN` to the platform release tag the customer runs, and list any
   optional feature they must not get in `DISABLED_FEATURES` (deny-list; unlisted features are
   deployed). Known features: `avq`, `quicktest`, `ai-test`, `virtual-scripts`.
4. **`customer.local.conf`** (git-ignored): the deploy targets, `role:alias` — `server:`, `frontend:`,
   `host:`; or pass `--targets` on the command line.
5. **Branding**: edit `frontend/.env.production` (name, tagline, title, logo URL or `/brand/<file>`,
   hidden/disabled nav entries) and drop assets in `frontend/public/brand/`. Everything here is
   baked into the public bundle — never put keys or tokens in it. See
   `docs/get-started/branding.md` in the platform for the full variable list.
6. **Customer files**: scripts, dashboards, identity maps, nginx confs at their platform-relative
   paths.
7. **Versioning**: `git config core.hooksPath .githooks` once per clone; every commit bumps
   `VERSION.txt`. The deployed system shows `<platform>/<overlay>`, e.g.
   `main-2026.09.02-8549/prod-2026.09.02-8553`, in the footer and in every host's ping.
8. **Before the first deploy to an existing machine**: copy its hand-placed frontend config
   (`frontend/public/favicon.ico`, `logo.png`, `branding.json`, `vite.config.local.json`) into the
   overlay if the customer wants to keep them; the overlay is authoritative for those paths.
9. **Deploy** from the platform's deploy host:

   ```bash
   git clone git@github.com:AngelStreetCorp/vpt-customer-<codename>.git ~/vpt-customer-<codename>
   bash ~/deploy_customer.sh ~/vpt-customer-<codename> --dry-run     # shows every change first
   bash ~/deploy_customer.sh ~/vpt-customer-<codename>
   ```

10. **Release**: moving the customer forward = change `PIN` to the next platform tag (read the
    release note's Features section first and add anything unwanted to `DISABLED_FEATURES`),
    commit, tag the overlay with its `VERSION.txt` value, deploy to the customer's test host,
    then to prod.

## What never goes in an overlay

- Backend `.env` files (database, storage and API keys). They stay on the machines; the deploy
  never reads, pushes or deletes them.
- The VM's `frontend/.env` (environment URLs and tokens). The overlay only ships
  `frontend/.env.production`, which Vite layers on top of it at build time.
- Platform source code. If a customer needs a code change, it goes to the platform behind
  configuration, or becomes an optional feature.

## This demo

`frontend/.env.production` brands the platform as "Demo Customer" with `brand/logo.svg`; no
feature is disabled; `PIN` follows the platform's integration branch. The machines it deploys to
are in `customer.local.conf` on the deploy host, not here.

## Deploying without GitHub access

If the customer's machines cannot reach GitHub, build the merged tree on a machine that has both
clones and ship one archive:

    bash ~/virtualpytest/setup/proxmox/node/build_customer_bundle.sh ~/vpt-customer-<codename> --out ~/bundles
    # -> ~/bundles/vpt-<codename>-<platform>_<overlay>.tar.gz (+ .sha256); no .git, no .env, unpacks to virtualpytest/

On the customer's storage VM: `rm -rf ~/virtualpytest && tar -xzf vpt-*.tar.gz -C ~`, check
`~/virtualpytest/BUNDLE_MANIFEST.txt`, then run their usual `update_core.sh` (refresh it once from
the bundle's `setup/proxmox/node/update_core.sh`, keeping their VM list). It skips the git step,
pushes the tree and runs `reconcile_feature_units.sh` on every host; the footer must show
`<platform>/<overlay>`. Details: `setup/proxmox/node/DEPLOY_CUSTOMER.md`, section "Deploying
without GitHub access (bundle)".
