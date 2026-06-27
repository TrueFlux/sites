TrueFlux Sites
==============

This repo contains the `Caddyfile` that hosts the shared configuration for TrueFlux' static sites, along w/ the individual production `Caddyfile`s which import that shared config. On push, the configs are deployed to production, but for local use, this repo is intended to be added as a Git submodule to whatever repo contains the code for that specific static site, w/ its development `Caddyfile` importing the one in this nested repo.

There's a main `Caddyfile`, used in production, that imports the common snippet, then imports the production site-specific Caddyfiles which themselves use the snippet in their own site-specific snippet which they then use in a production config, then there are development `Caddyfile`s for each site in their respective repos, which also import the common snippet and the snippet for the given site from this repo, intended to be nested as a Git submodule.

So, for a static site example.com, where directories are relative to the site repo and `caddy/` is this repo as a submodule:

`caddy/Caddyfile` (the main production `Caddyfile` that imports the common snippet, then all the site-specific `Caddyfile`s in production that use it):

```
import snippets/common/trueflux-static/Caddyfile
import snippets/sites/*/Caddyfile
import sites/*/Caddyfile
```

`caddy/snippets/sites/example.com/Caddyfile` (the site-specific `Caddyfile` that defines the site snippet which imports the common snippet):

```
(example.com) {
	import trueflux-static
}
```

`caddy/sites/example.com/Caddyfile` (the site-specific `Caddyfile` that defines the production site config which imports the site snippet and defines any production-specific config):

```
example.com {
	import example.com
}
```

`Caddyfile` (the development `Caddyfile` that 1. imports the common snippet, then the site-specific snippet, and 2. defines the development site config which imports the site snippet and defines any development-specific config):

```
import caddy/snippets/trueflux-static/Caddyfile
import caddy/snippets/sites/example.com/Caddyfile

example.localhost {
	import example.com
}
```
