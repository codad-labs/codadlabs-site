# codadlabs-site

The CoDad Lab umbrella site, served at `https://codadlabs.art`.

Plain HTML and one stylesheet. No build step, no JavaScript.

## Layout

| Path | Page |
|---|---|
| `index.html` | Home |
| `about/` | About us |
| `portfolio/` | Portfolio index |
| `portfolio/cielito-lindo/` | Cielito Lindo, linking out to `cielito.codadlabs.art`, with a looping video of the 3D model |
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
files straight from a clone at `/srv/codadlabs-site`. There is no
service to run and no build to do.

The setup below happens once. The VM, its static IP, and the 80/443
firewall rule already exist from the promo server's setup (see
`deploy/gcp/README.md` in `the-observatory`).

### 1. Push this repo to GitHub

Make the repo public. Everything in it is public on the site anyway,
and a public repo lets the VM clone and pull without a deploy key.

```bash
git init
git add .
git commit -m "Initial site skeleton"
gh repo create codad-labs/codadlabs-site --public --source=. --push
```

### 2. Add the DNS records

At Squarespace (Domains, `codadlabs.art`, DNS, DNS Settings, Custom
records), add:

| Type | Name | Data |
|---|---|---|
| `A` | `@` | the VM's static IP |
| `A` | `www` | the VM's static IP |

Before adding them, check the existing records for any `@` A record or
`www` CNAME pointing at Squarespace. Delete those. With two `@` records,
browsers pick one at random and half your visitors land on a Squarespace
parking page. Leave the `cielito` record and the Domain Connect and
email records alone.

Look up the static IP in Cloud Shell:
```bash
gcloud compute addresses describe observatory-promo-ip --region=us-west1 --format='value(address)'
```

Confirm both names resolve before touching Caddy, so Let's Encrypt
does not fail and count against its rate limit:
```bash
dig codadlabs.art A +short
dig www.codadlabs.art A +short
```
Both should print the static IP.

### 3. Clone the repo on the VM

SSH to the VM, then:
```bash
sudo mkdir -p /srv/codadlabs-site
sudo chown "$USER" /srv/codadlabs-site
git clone https://github.com/codad-labs/codadlabs-site /srv/codadlabs-site
```

Caddy runs as its own `caddy` user. Git creates folders as 755 and
files as 644, so Caddy can read everything without further changes.

### 4. Add the site blocks to the Caddyfile

Edit the file in place, so the existing `cielito` block and its
password hash stay untouched:
```bash
sudo nano /etc/caddy/Caddyfile
```

Add these two blocks above the `cielito.codadlabs.art` block:
```
codadlabs.art {
    root * /srv/codadlabs-site
    respond /.git/* 404
    file_server
}

www.codadlabs.art {
    redir https://codadlabs.art{uri} permanent
}
```

The finished file should look like this, with your real hash in place
of `<existing hash>`:
```
codadlabs.art {
    root * /srv/codadlabs-site
    respond /.git/* 404
    file_server
}

www.codadlabs.art {
    redir https://codadlabs.art{uri} permanent
}

cielito.codadlabs.art {
    @orders path /print /print/* /api/orders /api/orders/*
    basic_auth @orders {
        chris <existing hash>
    }
    reverse_proxy localhost:8000
}
```

Check the file, then reload:
```bash
caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
```

A reload keeps the promo server online. Only a typo that fails
validation would take it down, which is why validation runs first.

`file_server` does not list directories, and it serves `index.html`
for a folder request, so `/about/` returns `about/index.html`.

The `respond /.git/* 404` line matters: the clone's `.git` folder sits
inside the served directory, and without that line anyone could
download the repo's history from the site.

### 5. Confirm the certificates

```bash
sudo journalctl -u caddy --no-pager -o cat | grep -i "certificate obtained"
```
Look for one line each for `codadlabs.art` and `www.codadlabs.art`.

From your laptop:
```bash
curl -sI https://codadlabs.art/ | head -1
curl -sI https://www.codadlabs.art/ | grep -i location
curl -s -o /dev/null -w '%{http_code}\n' https://codadlabs.art/.git/config
```
Expect `HTTP/2 200`, a `location: https://codadlabs.art/` header, and
`404`.

If a certificate fails, the "Troubleshooting HTTPS" section of
`deploy/gcp/README.md` in `the-observatory` covers the problems from
the first setup.

## Redeploying after a change

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
