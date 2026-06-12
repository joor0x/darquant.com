# Darquant — Landing page

Static waitlist landing page for **darquant.com**, hosted on GitHub Pages.

## Files
- `index.html` — the landing page (this is what GitHub Pages serves).
- `darquant-landing.html` — original source copy (not served; can be deleted).
- `darquant-logo-dark.svg`, `darquant-mark.svg` — brand assets.
- `CNAME` — custom domain for GitHub Pages (`darquant.com`).
- `.nojekyll` — disables Jekyll processing.

## 1. Set up the waitlist (Formspree)
1. Sign up at https://formspree.io with `poriol@gmail.com`.
2. Create a new form → copy its endpoint, e.g. `https://formspree.io/f/abcdwxyz`.
3. In `index.html`, replace `YOUR_FORM_ID` in the form `action` with your real id.
4. First real submission triggers a one-time Formspree confirmation email — click it.

## 2. Deploy to GitHub Pages
```bash
git init
git add .
git commit -m "Darquant landing page"
git branch -M main
git remote add origin https://github.com/<your-user>/darquant.com.git
git push -u origin main
```
Then in the repo: **Settings → Pages → Build from branch → `main` / root**.

## 3. Point the domain (Cloudflare DNS)
In the Cloudflare dashboard for `darquant.com`, add these DNS records:

| Type  | Name | Value                         | Proxy        |
|-------|------|-------------------------------|--------------|
| A     | @    | 185.199.108.153               | DNS only (grey cloud) |
| A     | @    | 185.199.109.153               | DNS only |
| A     | @    | 185.199.110.153               | DNS only |
| A     | @    | 185.199.111.153               | DNS only |
| CNAME | www  | `<your-user>.github.io`       | DNS only |

> Use **DNS only** (grey cloud) at first so GitHub can issue its HTTPS cert.
> Once GitHub Pages shows the cert as issued, you may switch the proxy back on if desired.

Finally, in **Settings → Pages**, set the custom domain to `darquant.com` and tick **Enforce HTTPS** (after the cert is issued).
