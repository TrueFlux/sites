TrueFlux Sites
==============

This repo holds everything the TrueFlux static sites share:

  - the production Caddy config for the static server — on push to
    `master`, CI deploys it and reloads Caddy
  - the tooling each Zola site pulls in as its `caddy/` submodule — dev and
    deploy scripts, the CI deploy workflow, and shared template partials

Each site's own README only covers what's specific to that site, and links
here for the rest.

  - [Working on a site](#working-on-a-site) — setup, serving locally, deploying
  - [Adding a new site](#adding-a-new-site)
  - [How it works](#how-it-works)

## Working on a site

### Dependencies

  - [Zola](https://getzola.org) **0.22.1** to build the site. Not 0.23 —
    that moved to Tera 2, which drops the macro/import syntax the sites'
    templates use, so `brew install zola` (now 0.23) won't build them.
    Install the 0.22.1 release binary somewhere on your `PATH`:

    ```sh
    mkdir -p ~/bin
    curl -sSfL https://github.com/getzola/zola/releases/download/v0.22.1/zola-v0.22.1-aarch64-apple-darwin.tar.gz | tar xz -C ~/bin
    ```

    (`x86_64-apple-darwin` on Intel Macs.) If you already have 0.22.1 from
    Homebrew, `brew pin zola` stops it being upgraded.
  - [Caddy](https://caddyserver.com), which serves the site even in
    development. Config should work as far back as Caddy v2.6.2, the version
    Debian 13 (Trixie) packages, which is what the server runs.
  - [Rsync](https://rsync.samba.org) v3.1+, only for deploying manually with
    `bin/deploy` (it uses `--chown` and numeric `--chmod` modes; macOS ships
    2.6.9).
  - [Git LFS](https://git-lfs.com), only for sites that use it (their README
    says so). Run `git lfs install` once in the repo after cloning.

```sh
brew install caddy rsync
```

Clone with the submodule, or initialise it in an existing clone:

```sh
git clone --recurse-submodules <repo>
git submodule update --init -- caddy
```

### Serving locally

From the site's root:

```sh
bin/dev
```

builds the site and serves it at `http://localhost:3000` — no `/etc/hosts`
entry or `caddy trust` step needed. Pass a different `host:port` if 3000 is
taken (`bin/dev localhost:4000`).

`bin/dev` only builds once at startup. Caddy keeps serving, so rebuild after
changes with:

```sh
zola build --base-url http://localhost:3000
```

Match the `host:port` you're serving on, and keep the `http://` — without
the scheme, `resize_image` and multilingual URLs come out malformed.

### Deploying

Push to the site's default branch. GitHub Actions builds the site and
deploys it to the domain in `zola.toml`'s `base_url`.

To deploy by hand instead (needs SSH access to the server):

```sh
zola build  # with the production base_url from zola.toml
bin/deploy  # rsyncs public/ to the server with the right permissions
```

### Contact forms

Sites with a contact form post it to `https://trueflux.agency/api/contact_form`
(TrueFlux's `Api::ContactFormController`), which emails the enquiry on.

  - The form's hidden `token` has to be registered in TrueFlux's encrypted
    credentials, under `contact_form_tokens` (with the email and redirect
    URLs for that site). An unregistered token is rejected.
  - The form includes the shared Cloudflare Turnstile widget — in the site's
    template, `{% import "turnstile.html" as turnstile %}` and
    `{{ turnstile::widget() }}`. The widget only renders on domains listed
    in its Cloudflare dashboard config, so a site's domain needs adding
    there.

### Google Analytics

Set the site's GA4 measurement ID in `zola.toml`:

```toml
[extra]
ga_measurement_id = "G-XXXXXXXXXX"
```

and include the shared tag in `base.html`'s `<head>`:

```
{% include "google_analytics.html" %}
```

It's only rendered by `zola build`, so local visits aren't counted.

### Changing shared tooling

Make the change here and push it, then bump each site's submodule to pick it
up:

```sh
git -C caddy pull origin master
git add caddy
```

The CI workflow is the exception — sites call it at `@master`, so changes to
it apply on each site's next deploy without a bump.

## Adding a new site

On the server side:

1. Add `sites/<domain>/Caddyfile`:

   ```
   example.com {
   	import trueflux-static
   }
   ```

2. Push — CI will deploy the new config and reload Caddy.

3. On the server, ensure the web root directory is writable by the `deploy`
   user (once — already done for the current server):

   ```sh
   sudo chown root:caddy /usr/share/caddy
   sudo chmod 775 /usr/share/caddy
   ```

   Deploys create `/usr/share/caddy/<domain>/` via rsync's `--mkpath`.

In the site's repo:

1. Add this repo as the `caddy` submodule, and symlink in the shared scripts
   and partials:

   ```sh
   git submodule add ../sites.git caddy
   ln -s ../caddy/bin/dev bin/dev
   ln -s ../caddy/bin/deploy-site bin/deploy
   ln -s ../caddy/templates/turnstile.html templates/turnstile.html
   ln -s ../caddy/templates/google_analytics.html templates/google_analytics.html
   ```

2. Set `base_url` in `zola.toml` to the site's own domain — it's where
   `bin/deploy` deploys to, so it can't be shared with another site.

3. Add `.github/workflows/deploy.yml`:

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

   and add the `deploy` user's private key as that repo secret.

4. Use the same `.gitignore` as the other sites (Git won't follow a
   symlinked one):

   ```
   /public
   /static/processed_images
   .DS_Store
   ```

## How it works

```
Caddyfile                          # Main production Caddyfile — imports the common snippet and all site configs
snippets/common/trueflux-static/   # Shared server config (file_server, clean URLs, error handling, compression)
sites/<domain>/Caddyfile           # Per-site production config — imports trueflux-static
bin/deploy                         # Deploys this repo's Caddy config to the server
bin/dev                            # Sites' local dev script
Caddyfile.dev                      # Local dev config bin/dev serves sites with
bin/deploy-site                    # Sites' deploy script
templates/                         # Shared Zola partials (Turnstile, Google Analytics)
.github/workflows/deploy-zola-site.yml  # Reusable CI deploy workflow sites call
.github/known_hosts                # The static server's SSH host key
```

### Local development

`bin/dev` runs `zola build --base-url http://<host>` and then
`caddy run --watch` with `Caddyfile.dev`, which serves `public/` on
`$CADDY_HOST`:

```
http://{$CADDY_HOST:localhost:3000} {
	import trueflux-static
	root public
}
```

The `http://` prefix matters — Caddy treats `localhost` as an internal
trusted name and gives it automatic HTTPS regardless of port, so without it
`bin/dev`'s build (plain `http://` base URL) and what Caddy actually serves
would disagree.

### Deploying

`bin/deploy-site` rsyncs the built `public/` to the static server, taking the
domain from `base_url` in the site's `zola.toml` — that's both the SSH host
and the `/usr/share/caddy/<domain>` directory it's served from, so every site
needs its own domain. In CI (`CI=true`) it connects as the `deploy` user.

The reusable `deploy-zola-site.yml` workflow pins the Zola version, checks
out the site (with LFS and submodules), builds it and runs its `bin/deploy`.
The server's host key is in `.github/known_hosts`, under a `*` pattern since
every site's domain points at the same server.
