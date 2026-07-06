TrueFlux Sites
==============

This repo contains the Caddy configuration for all TrueFlux static sites. On push to `master`, CI deploys the full configuration to production and reloads Caddy.

## Structure

```
Caddyfile                          # Main production Caddyfile — imports the common snippet and all site configs
snippets/common/trueflux-static/   # Shared server config (file_server, clean URLs, error handling, compression)
sites/<domain>/Caddyfile           # Per-site production config — imports trueflux-static
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

Each site repo includes this repo as a `caddy/` submodule, used only for the local development `Caddyfile`. See the individual site repo READMEs for setup instructions.
