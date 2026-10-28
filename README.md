# codadLabs-site

The CoDad Labs umbrella site, served at `https://codadLabs.art`.

Plain HTML and one stylesheet. No build step, no JavaScript.

## Layout

| Path | Page |
|---|---|
| `index.html` | Home |
| `about/` | About us |
| `portfolio/` | Portfolio index |
| `portfolio/cielito-lindo/` | Cielito Lindo, linking out to `cielito.codadLabs.art`, with a looping video of the 3D model |
| `portfolio/max-loads/` | Max Loads |
| `merch/` | Merch ("coming soon") |
| `events/` | Events, edited by hand |
| `style.css` | Shared styles |
| `images/` | Logo and photos |

Each page lives in its own folder as `index.html`, so URLs read
`/about/` instead of `/about.html`.

Donate and Instagram are external links in the nav, not pages.

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
files straight from a clone at `/srv/codadLabs-site`. There is no
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

The user, configuration, and key match `inv sync-db` in
`the-observatory`. Without `christoph_rico@`, gcloud logs in as your
laptop username, which does not own the clone, and git refuses to pull.

`--ff-only` refuses to pull if the VM's clone has diverged, which
should never happen since you never edit files on the VM.

Reload Caddy only after editing the Caddyfile itself.
