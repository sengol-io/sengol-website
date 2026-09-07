# Deploying sengol.io

The site is static (HTML, CSS, fonts, images; no build step) and is hosted on
Cloudflare Pages as the project `sengol-website`, connected to this GitHub
repository through Cloudflare's Git integration. Cloudflare deploys every
push to `main` to production and every pull request to a preview URL. No API
token, GitHub secret, or GitHub Actions workflow is involved.

Files that matter:

| File | Purpose |
|---|---|
| `wrangler.toml` | Live build configuration read by Cloudflare on every Git build: names the Pages project and sets the deploy directory to the repo root. See the note at the end of section 1. |
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

`wrangler.toml` is not just documentation. Because it contains
`pages_build_output_dir`, Cloudflare detects it during every Git build and
treats it as the source of truth for the settings it defines: the build
output directory today, and any compatibility date, compatibility flags,
environment variables, or bindings that are ever added to it. Those fields
become read-only in the dashboard while the file is present, and a
dashboard-only change to them will be overridden on the next build. The two
must agree: `pages_build_output_dir = "."` in the file is the same thing as
Build output directory `/` in the dashboard. To change a setting the file
covers, change the file and push. Settings the file does not cover (build
command, production branch, preview access, notifications) still live under
the project -> Settings.

## 2. Move the sengol.io custom domain

Status: done. On 2026-09-07 the operator attached `sengol.io` to
`sengol-website`, confirmed with `curl -I` that `https://sengol.io` and
`https://www.sengol.io` returned 200 with the `_headers` values and the same
asset etags as `sengol-website.pages.dev`, and then deleted `sengol-landing`.
Nothing in this repository performs that cutover; it is a dashboard
procedure, kept here for the next time a hostname has to move between Pages
projects. If you are reading this on a fresh account or after a project has
been recreated, re-run section 3 before trusting this status line.

The old project `sengol-landing` owned the `sengol.io` custom domain.
Cloudflare only lets one Pages project hold a given hostname, and it offers
no atomic transfer between projects, so the order matters and there are two
possible paths.

1. Confirm the new project serves the site at its `*.pages.dev` URL.
2. List every hostname under `sengol-landing` -> Custom domains. Each one
   (`sengol.io`, and `www.sengol.io` if present) has to be moved and verified
   individually.
3. Workers & Pages -> `sengol-website` -> Custom domains -> Set up a custom
   domain -> `sengol.io`.
   - If Cloudflare accepts the attach, there is no interruption: the hostname
     switches to the new project as soon as its status shows Active.
   - If Cloudflare refuses because the hostname is attached to another
     project, you have to detach first: `sengol-landing` -> Custom domains ->
     remove `sengol.io`, then immediately re-add it on `sengol-website`.
     Between the removal and the new association showing Active, requests
     to that hostname get a Cloudflare error page. In practice this is
     seconds to about a minute, but it is a real gap, so do it at a quiet
     time and have both browser tabs open before you start.
   Repeat for `www.sengol.io`.
4. Because `sengol.io` is a zone in the same Cloudflare account, the DNS
   record is a proxied CNAME pointing at the project's `pages.dev` hostname.
   Cloudflare updates that record automatically when the custom domain is
   attached. You do not need to edit DNS by hand; if you do look, it should
   read `sengol.io CNAME sengol-website.pages.dev` (proxied).
5. Wait until every moved hostname shows Active under `sengol-website` ->
   Custom domains, then run the verification in section 3 for each of them.
6. Only after every hostname is confirmed to be served by the new project,
   delete `sengol-landing` (its Settings -> General -> Delete project) so a
   hostname cannot be re-attached to it by mistake. Deleting it while a
   hostname is still pending or still attached to it takes that hostname
   down.

## 3. Verify

After the first deploy and after the custom domain move, check every
hostname, not just the apex:

```sh
curl -I https://sengol.io
curl -I https://www.sengol.io
curl -sI http://sengol.io | head -1
```

Expect `HTTP/2 200` on the first two (or a 301 to the apex from `www`, if a
redirect rule is configured) and a 301 to HTTPS on the third. Each 200 must
carry the headers defined in `_headers`:

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
(check the build log under the project -> Deployments) or that hostname is
still pointing at the old project (check Custom domains on both projects).
A Cloudflare error page on one hostname while the other works means that
hostname's custom-domain association is still pending or was never moved.

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
