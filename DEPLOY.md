# Deploying sengol.io

The site is static (HTML, CSS, fonts, images; no build step) and is hosted on
Cloudflare Pages as the project `sengol-website`. Every push to `main` deploys
to production through `.github/workflows/deploy-cloudflare.yml`; every pull
request gets a preview deploy with the URL posted as a PR comment.

Files that matter:

| File | Purpose |
|---|---|
| `wrangler.toml` | Names the Pages project and sets the deploy directory to the repo root. |
| `_headers` | The only place response headers are defined. |
| `.github/workflows/deploy-cloudflare.yml` | Production and preview deploys. |
| `CNAME`, `.nojekyll` | Leftovers from GitHub Pages; harmless on Cloudflare, kept so the repo still works there as a fallback. |

There is no `_redirects` file. Add one only if a URL actually needs to move.

## 1. Create the Pages project

Do this once. Pick one of the two options; do not do both.

### Option A: Direct Upload (recommended, matches the workflow)

The GitHub workflow uploads files itself, so the Pages project must be a
Direct Upload project (not connected to Git in the Cloudflare dashboard).

From a machine with the repo checked out and Node 18+ installed:

```sh
npx wrangler login
npx wrangler pages project create sengol-website --production-branch=main
npx wrangler pages deploy . --project-name=sengol-website --branch=main
```

The last command does the first production deploy and prints a
`sengol-website.pages.dev` URL. Check that URL renders the site before moving
on. After this, the workflow handles every further deploy.

### Option B: Dashboard Git integration

Cloudflare dashboard -> Workers & Pages -> Create -> Pages -> Connect to Git ->
choose `sengol-io/sengol-website`, production branch `main`, framework preset
None, build command empty, build output directory `/`.

If you choose this option, Cloudflare builds on its own on every push and the
GitHub workflow is redundant. Either delete the workflow or leave it disabled
(Actions -> Deploy to Cloudflare Pages -> Disable workflow), otherwise the
project will receive two deploys per push. Wrangler cannot deploy to a
Git-connected project.

## 2. Move the sengol.io custom domain without downtime

The old project (from the archived `archive-sengol-landing` repo) currently
owns the `sengol.io` custom domain. Cloudflare only lets one Pages project hold
a given hostname, so the order matters.

1. Confirm the new project serves the site at its `*.pages.dev` URL.
2. Cloudflare dashboard -> Workers & Pages -> `sengol-website` -> Custom
   domains -> Set up a custom domain -> `sengol.io`. If Cloudflare refuses
   because the hostname is attached to another project, go to the old
   project -> Custom domains and remove `sengol.io` there, then immediately
   re-add it on `sengol-website`. Do the same for `www.sengol.io` if the old
   project had it.
3. Because `sengol.io` is a zone in the same Cloudflare account, the DNS
   record is a proxied CNAME pointing at the project's `pages.dev` hostname.
   Cloudflare updates that record automatically when the custom domain is
   attached. You do not need to edit DNS by hand; if you do look, it should
   read `sengol.io CNAME sengol-website.pages.dev` (proxied).
4. Wait for the custom domain status to show Active (usually under a minute),
   then run the verification in section 4.
5. Only after `https://sengol.io` is confirmed to be served by the new
   project, delete the old Pages project so it cannot be re-attached by
   mistake.

The cut-over is a single record change on Cloudflare's edge, so there is no
propagation delay and no window where the site is down.

## 3. GitHub repository secrets

Settings -> Secrets and variables -> Actions -> New repository secret:

| Secret | Value |
|---|---|
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare dashboard -> Workers & Pages -> Overview, right-hand column "Account ID". |
| `CLOUDFLARE_API_TOKEN` | A token created as described below. |

Create the token at Cloudflare dashboard -> My Profile -> API Tokens ->
Create Token -> Create Custom Token with exactly this permission:

- Account -> Cloudflare Pages -> Edit

Scope Account Resources to the one account that owns the project. Nothing
else is needed: no zone permissions, no user permissions. Set a TTL if you
want the token to expire, and rotate it by creating a new token and replacing
the secret.

The workflow also passes the built-in `GITHUB_TOKEN` to the action so that
each deploy shows up under the repository's Deployments tab; that token is
provided automatically and is not something you add.

## 4. Verify

After the first workflow run on `main` (Actions tab, "Deploy to Cloudflare
Pages") and after the custom domain move:

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

If those headers are missing, the deploy did not include `_headers` (check
the workflow log) or the domain is still pointing at the old project (check
Custom domains on both projects).

## Manual deploy from a laptop

If GitHub Actions is unavailable, the same command the workflow runs works
locally after `npx wrangler login`:

```sh
npx wrangler pages deploy . --project-name=sengol-website --branch=main
```

## What gets uploaded

`wrangler pages deploy .` uploads everything in the repo root except `.git`,
`node_modules` and `.DS_Store`. That includes this file, `wrangler.toml` and
the workflow file. All of it is already public in the repository, so nothing
sensitive is exposed; it is simply reachable at, for example,
`https://sengol.io/DEPLOY.md`. If that bothers you, add `_redirects` rules
returning 404 for those paths.
