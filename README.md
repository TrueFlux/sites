TrueFlux Sites
==============

This repo contains the Caddy configuration for all TrueFlux static sites. On push to `master`, CI deploys the full configuration to production and reloads Caddy.

## Structure

```
Caddyfile                          # Main production Caddyfile — imports the common snippet and all site configs
snippets/common/trueflux-static/   # Shared server config (file_server, clean URLs, error handling, compression)
sites/<domain>/Caddyfile           # Per-site production config — imports trueflux-static
bin/dev                            # Shared local dev script — see "Local development"
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

## Local development

Each site repo includes this repo as a `caddy/` submodule, used for the local
development `Caddyfile` and for `bin/dev`, the shared dev script — symlink it
in:

```sh
ln -s ../caddy/bin/dev bin/dev
```

`bin/dev` builds the site once and runs `caddy run --watch` against
`$CADDY_HOST`, which the site's own local `Caddyfile` should declare with the
same generic default and an explicit `http://` prefix:

```
http://{$CADDY_HOST:localhost:3000} {
	import trueflux-static
	root public
}
```

The `http://` prefix matters — Caddy treats `localhost` as an internal
trusted name and gives it automatic HTTPS regardless of port, so without it
`bin/dev`'s build (plain `http://` base URL) and what Caddy actually serves
would disagree. The default (`localhost:3000`) needs no `/etc/hosts` entry
or `caddy trust` step. Pass a different `host:port` as `bin/dev`'s first
argument if 3000 is taken.

Bump the site's `caddy` submodule pointer to pick up changes made here.
