# Deploying sengol.io

The site is static (HTML, CSS, fonts, images; no build step) and is hosted on
Cloudflare Pages as the project `sengol-website`, connected to this GitHub
repository through Cloudflare's Git integration. Cloudflare deploys every
push to `main` to production and every pull request to a preview URL. No API
token, GitHub secret, or GitHub Actions workflow is involved.

Files that matter:

| File | Purpose |
|---|---|
| `wrangler.toml` | Names the Pages project and sets the deploy directory to the repo root. |
| `_headers` | The only place response headers are defined. |
| `CNAME`, `.nojekyll` | Leftovers from GitHub Pages; harmless on Cloudflare, kept so the repo still works there as a fallback. |

There is no `_redirects` file. Add one only if a URL actually needs to move.

## Why a new project rather than editing the old one

The existing Pages project is `sengol-landing`, connected to the repository
`sengol-io/sengol-landing` (since renamed and archived as
`archive-sengol-landing`). Cloudflare Pages cannot change which Git
repository a project is connected to, so the old project cannot be pointed
at this repo. The replacement is: create `sengol-website` connected to this
repo, move the custom domain across, then delete `sengol-landing`.

Settings on `sengol-landing` worth carrying over, all under the new
project -> Settings once it exists:

| Old setting | Value on `sengol-landing` | On the new project |
|---|---|---|
| Production branch | `main` | Same (set during creation). |
| Automatic deployments | Enabled | Default, leave on. |
| Build command / output / root | all empty | Same; output directory `/`. |
| Build comments | Enabled | Default, leave on. |
| Build cache | Disabled | Default for a no-build project. |
| Build system version | 3 | Default. |
| Preview access | Restricted by a Cloudflare Access policy | Not copied automatically. Re-enable under Settings -> General -> Preview access -> Manage after the first deploy, otherwise preview URLs are public. |
| Notifications | whatever is configured | Not copied. Re-add under Settings -> General -> Notifications if any were in use. |
| Variables, bindings, Functions | none | Nothing to copy. |

## 1. Connect the repository

Do this once, in the Cloudflare dashboard.

1. Workers & Pages -> Create -> Pages -> Connect to Git.
2. If GitHub is not yet linked, Cloudflare opens the GitHub App install page.
   Install it on the `sengol-io` organisation and grant it access to
   `sengol-website` only. The app needs read access to the repo plus
   permission to post deployment statuses and pull request comments; that is
   the default set it asks for.
3. Select `sengol-io/sengol-website`, then Begin setup.
4. Settings:

   | Field | Value |
   |---|---|
   | Project name | `sengol-website` |
   | Production branch | `main` |
   | Framework preset | None |
   | Build command | leave empty |
   | Build output directory | `/` |

5. Save and Deploy. Cloudflare clones `main`, uploads the repo root, and
   prints a `sengol-website.pages.dev` URL. Open it and check the site
   renders before moving on.

From now on every push to `main` is a production deploy and every pull
request gets a preview deploy. The Cloudflare GitHub App posts the preview
URL as a check and a comment on the pull request; there is nothing else to
configure for that.

If you later want to change the build settings, they live under the project
-> Settings -> Builds & deployments. `wrangler.toml` in the repo is not read
by the Git integration; it is there for manual wrangler deploys and as
documentation of the project name and deploy directory.

## 2. Move the sengol.io custom domain without downtime

The old project `sengol-landing` currently owns the `sengol.io` custom
domain. Cloudflare only lets one Pages project hold a given hostname, so the
order matters.

1. Confirm the new project serves the site at its `*.pages.dev` URL.
2. Workers & Pages -> `sengol-website` -> Custom domains -> Set up a custom
   domain -> `sengol.io`. If Cloudflare refuses because the hostname is
   attached to another project, go to `sengol-landing` -> Custom domains and
   remove `sengol.io` there, then immediately re-add it on `sengol-website`.
   Do the same for `www.sengol.io` if `sengol-landing` had it.
3. Because `sengol.io` is a zone in the same Cloudflare account, the DNS
   record is a proxied CNAME pointing at the project's `pages.dev` hostname.
   Cloudflare updates that record automatically when the custom domain is
   attached. You do not need to edit DNS by hand; if you do look, it should
   read `sengol.io CNAME sengol-website.pages.dev` (proxied).
4. Wait for the custom domain status to show Active (usually under a minute),
   then run the verification in section 3.
5. Only after `https://sengol.io` is confirmed to be served by the new
   project, delete `sengol-landing` (its Settings -> General -> Delete
   project) so the hostname cannot be re-attached to it by mistake.

The cut-over is a single record change on Cloudflare's edge, so there is no
propagation delay and no window where the site is down.

## 3. Verify

After the first deploy and after the custom domain move:

```sh
curl -I https://sengol.io
```

Expect `HTTP/2 200` and the headers defined in `_headers`:

```
x-content-type-options: nosniff
referrer-policy: strict-origin-when-cross-origin
x-frame-options: SAMEORIGIN
permissions-policy: geolocation=(), microphone=(), camera=()
```

Then check that a hashed font asset carries the long-lived cache rule:

```sh
curl -I https://sengol.io/fonts/inter-latin.3100e775.woff2
```

Expect `cache-control: public, max-age=31536000, immutable`.

If those headers are missing, either the deploy did not include `_headers`
(check the build log under the project -> Deployments) or the domain is still
pointing at the old project (check Custom domains on both projects).

## Manual deploy from a laptop

Normally unnecessary. If the Git integration is broken or you need to push a
fix while GitHub is down, a Git-connected project can still receive a direct
upload from wrangler after logging in with your own Cloudflare account:

```sh
npx wrangler login
npx wrangler pages deploy . --project-name=sengol-website --branch=main
```

This uses your browser login, not an API token, and reads the project name
and deploy directory from `wrangler.toml`. The next push to `main` replaces
the manual deploy.

## What gets uploaded

The deploy directory is the repo root, so everything except `.git` is
uploaded: that includes this file and `wrangler.toml`. Both are already
public in the repository, so nothing sensitive is exposed; they are simply
reachable at, for example, `https://sengol.io/DEPLOY.md`. If that bothers
you, add `_redirects` rules returning 404 for those paths.
