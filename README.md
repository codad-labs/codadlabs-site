# codadlabs-site

The CoDad Labs umbrella site, served at `https://codadlabs.art`.

Plain HTML and one stylesheet. No build step, no JavaScript.

## Layout

| Path | Page |
|---|---|
| `index.html` | Home |
| `about/` | About us |
| `portfolio/cielito-lindo/` | Cielito Lindo, linking out to `cielito.codadlabs.art`, with a looping video of the 3D model |
| `portfolio/max-loads/` | Max Loads |
| `merch/` | Merch ("coming soon") |
| `desktop-cielito/guide/` | User guide for the Desktop Cielito, linked from the device's companion page and from the `/g` short link |
| `events/` | Events, edited by hand |
| `style.css` | Shared styles |
| `images/` | Logo and photos |

Each page lives in its own folder as `index.html`, so URLs read
`/about/` instead of `/about.html`.

Donate and Instagram are external links in the nav, not pages. The nav's
Portfolio entry is a `<details>` dropdown listing each piece; `/portfolio/`
itself redirects to the home page in the Caddyfile.

**Media.** Keep images no wider than about 1400px and use WebP. Use a
muted looping MP4 instead of a GIF. Give every `<img>` and `<video>` its
`width` and `height` so the page does not jump while it loads.

## Editing

**The nav repeats on every page.** A change to the nav means editing
every `index.html`. With six pages this costs less than adding a build
step or JavaScript includes.

**Placeholders.** Search for `PLACEHOLDER` (copy to rewrite) and `TODO`
(links and images to fill in):

```bash
grep -rn -e PLACEHOLDER -e TODO --include='*.html' .
```

**Events.** Copy an `<article>` block in `events/index.html` into
"Upcoming". Move or delete it after the date passes.

## Local preview

```bash
python3 -m http.server 8080
```

Then open `http://localhost:8080`. Pages use root-relative paths
(`/style.css`), so open them through the server, not as `file://` URLs.

## Deploying

The site runs on the same VM as the Cielito promo server
(`observatory-promo`), behind the same Caddy. Caddy serves this repo's
files straight from a clone at `/srv/codadlabs-site`. There is no
service to run and no build to do.

Push from your laptop, then pull on the VM. Caddy reads files from
disk on each request, so it needs no reload.

```bash
git push
gcloud compute ssh christoph_rico@observatory-promo --zone=us-west1-a \
  --configuration=observatory \
  --ssh-key-file ~/.ssh/observatory_gce \
  --command='git -C /srv/codadlabs-site pull --ff-only'
```

Or from the VM itself:

```bash
update_codad
```

The user, configuration, and key match `inv sync-db` in
`the-observatory`. Without `christoph_rico@`, gcloud logs in as your
laptop username, which does not own the clone, and git refuses to pull.

`--ff-only` refuses to pull if the VM's clone has diverged, which
should never happen since you never edit files on the VM.

Reload Caddy only after editing the Caddyfile itself. The Caddyfile is
tracked in `the-observatory` at `deploy/gcp/Caddyfile`, because one file
serves both sites. It holds the `/g` short link to the Desktop Cielito
guide. See "Updating the Caddyfile" in `deploy/gcp/README.md` there.
