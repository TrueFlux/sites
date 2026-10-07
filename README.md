TrueFlux Sites
==============

This repo contains the Caddy configuration for all TrueFlux static sites. On push to `master`, CI deploys the full configuration to production and reloads Caddy.

## Structure

```
Caddyfile                          # Main production Caddyfile — imports the common snippet and all site configs
snippets/common/trueflux-static/   # Shared server config (file_server, clean URLs, error handling, compression)
sites/<domain>/Caddyfile           # Per-site production config — imports trueflux-static
bin/dev                            # Shared local dev script — see "Local development"
Caddyfile.dev                      # Local dev config bin/dev serves sites with
bin/deploy-site                    # Shared site deploy script — see "Deploying a site"
```

## Adding a new site

1. Add `sites/<domain>/Caddyfile`:

```
example.com {
	import trueflux-static
}
```

2. Push — CI will deploy the new config and reload Caddy.

3. On the server, ensure the web root directory is writable by the `deploy` user:

```sh
sudo chown root:caddy /usr/share/caddy
sudo chmod 775 /usr/share/caddy
```

Subsequent deploys from the site repo will create `/usr/share/caddy/<domain>/` automatically via rsync's `--mkpath`.

## Deploying a site

Each site symlinks in `bin/deploy-site` as its own `bin/deploy`:

```sh
ln -s ../caddy/bin/deploy-site bin/deploy
```

It rsyncs the built `public/` to the static server, taking the domain from
`base_url` in the site's `zola.toml` — that's both the SSH host and the
`/usr/share/caddy/<domain>` directory it's served from, so every site needs
its own domain. In CI (`CI=true`) it connects as the `deploy` user.

CI is the reusable `.github/workflows/deploy-zola-site.yml` workflow, which
pins the Zola version, builds the site and runs its `bin/deploy`. Each site's
own `.github/workflows/deploy.yml` just calls it on push, passing the site's
deploy key secret:

```yaml
name: Deploy site

on:
  push:
    branches: ['master']

jobs:
  deploy:
    uses: TrueFlux/sites/.github/workflows/deploy-zola-site.yml@master
    secrets:
      ssh_private_key: ${{ secrets.<SITE>_SSH_PRIVATE_KEY }}
```

The server's host key is in `.github/known_hosts`, under a `*` pattern since
every site's domain points at the same server.

## Local development

Each site repo includes this repo as a `caddy/` submodule, used for
`bin/dev`, the shared dev script — symlink it in:

```sh
ln -s ../caddy/bin/dev bin/dev
```

Run from the site's root, `bin/dev` builds the site once and runs
`caddy run --watch` with this repo's `Caddyfile.dev`, which serves `public/`
on `$CADDY_HOST` with an explicit `http://` prefix:

```
http://{$CADDY_HOST:localhost:3000} {
	import trueflux-static
	root public
}
```

Sites don't need a `Caddyfile` of their own.

The `http://` prefix matters — Caddy treats `localhost` as an internal
trusted name and gives it automatic HTTPS regardless of port, so without it
`bin/dev`'s build (plain `http://` base URL) and what Caddy actually serves
would disagree. The default (`localhost:3000`) needs no `/etc/hosts` entry
or `caddy trust` step. Pass a different `host:port` as `bin/dev`'s first
argument if 3000 is taken.

Bump the site's `caddy` submodule pointer to pick up changes made here.
