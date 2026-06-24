# Darquant — Landing page + blog

Static site for **darquant.com**, hosted on GitHub Pages:

| URL          | Served from                                     |
|--------------|-------------------------------------------------|
| `/`          | `index.html` — waitlist landing page            |
| `/blog/`     | Hugo blog, built from [`blog/`](blog/)          |

Both are built and published together by a single GitHub Actions workflow
([`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)) on every push
to `main`.

## Files
- `index.html` — the landing page (served at the site root).
- `darquant-landing.html` — original source copy (not served; can be deleted).
- `darquant-logo-dark.svg`, `darquant-mark.svg` — brand assets.
- `CNAME` — custom domain for GitHub Pages (`darquant.com`).
- `.nojekyll` — disables Jekyll processing.
- `blog/` — Hugo site (see [Blog](#blog) below).

## 1. Waitlist (Formspree) — configured
The form posts to `https://formspree.io/f/xnjyggvp` via AJAX (see the script at
the bottom of `index.html`), so the page never redirects. The first real
submission triggers a one-time Formspree confirmation email — click it to start
receiving signups. Free tier: 50 submissions/month.

## 2. Deploy to GitHub Pages (GitHub Actions)

Deployment is automated. **One-time setup:** in the repo go to
**Settings → Pages → Build and deployment → Source** and select
**GitHub Actions** (not "Deploy from a branch"). This is required — otherwise
GitHub keeps serving the raw `main` branch and ignores the Hugo build, so
`/blog/` would 404.

After that, every push to `main` triggers
[`.github/workflows/deploy.yml`](.github/workflows/deploy.yml), which:

1. Installs **Hugo extended** (version pinned via `HUGO_VERSION` in the workflow).
2. Builds the blog into `_site/blog/` (`hugo --source blog --minify`).
3. Copies the landing page + `CNAME` + `.nojekyll` + brand SVGs into `_site/`.
4. Uploads `_site/` and deploys it to GitHub Pages.

You can also trigger it manually from the **Actions** tab
(**Deploy site to GitHub Pages → Run workflow**). Watch the run there; when it
finishes the site is live at <https://darquant.com> and the blog at
<https://darquant.com/blog/>.

> **Keep Hugo in sync.** `HUGO_VERSION` in the workflow must match the version
> you use locally (`hugo version`). Bump it when you upgrade Hugo so local and
> CI builds stay identical.

## 3. Point the domain (Cloudflare DNS)
In the Cloudflare dashboard for `darquant.com`, add these DNS records:

| Type  | Name | Value                         | Proxy        |
|-------|------|-------------------------------|--------------|
| A     | @    | 185.199.108.153               | DNS only (grey cloud) |
| A     | @    | 185.199.109.153               | DNS only |
| A     | @    | 185.199.110.153               | DNS only |
| A     | @    | 185.199.111.153               | DNS only |
| CNAME | www  | `joor0x.github.io`            | DNS only |

> Use **DNS only** (grey cloud) at first so GitHub can issue its HTTPS cert.
> Once GitHub Pages shows the cert as issued, you may switch the proxy back on if desired.

Finally, in **Settings → Pages**, set the custom domain to `darquant.com` and
tick **Enforce HTTPS** (after the cert is issued).

## Blog

The blog lives in [`blog/`](blog/) and is a [Hugo](https://gohugo.io) site with
custom layouts (no external theme).

- `blog/hugo.toml` — config. `baseURL = "https://darquant.com/blog/"`, so every
  generated link (CSS, posts, RSS) is correctly prefixed with `/blog/`.
- `blog/content/posts/` — posts, one Markdown file each.
- `blog/layouts/` — templates (`_default/`, `partials/`).
- `blog/assets/css/darquant.css` — styles (minified + fingerprinted at build).
- `blog/static/` — files copied verbatim (logo/mark SVGs).

Build artifacts (`blog/public/`, `blog/resources/`, `blog/.hugo_build.lock`) are
git-ignored — CI rebuilds them, so don't commit them.

### Write a post

```bash
cd blog
hugo new content posts/my-post.md   # creates a draft
```

Edit the file, then drop `draft: true` (or build with `--buildDrafts`) when
ready to publish.

### Preview locally

```bash
cd blog
hugo server                         # http://localhost:1313/blog/
```

Push to `main` to publish — the workflow handles the rest.
