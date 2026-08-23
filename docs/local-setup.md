# Environment Setup

This is the canonical path for standing up the public demo stack (Saleor API + worker +
dashboard + storefront, behind Nginx/Certbot) on a fresh Linux VM. It reflects what actually
worked after debugging a live deployment — the gotchas below aren't hypothetical, they're
things that broke on the first attempt and why the fix looks the way it does.

Target: under 30 minutes on a clean machine.

## Prerequisites

- A Linux VM (tested on a 2 vCPU / 4GB KVM VPS) with Docker + Docker Compose installed
  (`curl -fsSL https://get.docker.com | sh`), and root/sudo access.
- A domain with four DNS `A` records already pointing at the VM's public IP: the bare domain
  (`@`), plus `api.`, `shop.`, `admin.`, `mail.` subdomains. Confirm with `nslookup` before
  starting — nothing below will work until DNS has propagated.
- Port 80/443 available for Nginx. If another app already runs on this box, it can coexist —
  just make sure it's not also claiming ports `8001`, `9000`, `3000`, `8025` on `127.0.0.1`
  (check with `sudo ss -tlnp`). If any of those are taken, remap the conflicting service in
  `infra/docker-compose.demo.yml`'s `ports:` section and update the matching Nginx
  `proxy_pass` to match.
- 4GB RAM is tight for this stack. A swap file is recommended:
  ```bash
  sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile
  sudo mkswap /swapfile && sudo swapon /swapfile
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  ```

## 1. Clone the repos

The storefront must be cloned as a **sibling** of this repo's own checkout — the compose
file's build context depends on that exact layout.

```bash
mkdir -p ~/chidambarams/apps && cd ~/chidambarams/apps
git clone <this repo's URL> chidambarams
git clone https://github.com/saleor/storefront.git
```

Result:
```
~/chidambarams/apps/
  chidambarams/   <- this repo (infra/docker-compose.demo.yml lives here)
  storefront/     <- build context only, not run directly
```

Note: `saleor-platform` is **not** needed. `infra/docker-compose.demo.yml` is self-contained
and pulls the official `ghcr.io/saleor/saleor` and `ghcr.io/saleor/saleor-dashboard` images
directly rather than building from source.

## 2. Create the secrets file

```bash
cd ~/chidambarams/apps/chidambarams
cp infra/.env.demo.example infra/.env.demo
```

Edit `infra/.env.demo` and fill in:

- `SECRET_KEY` — `openssl rand -base64 48`
- `POSTGRES_PASSWORD` — `openssl rand -hex 24`

**RSA_PRIVATE_KEY needs its own step** — Saleor requires it for JWT signing once
`DEBUG=False`, and it's the one value in this file that's genuinely easy to get wrong: it's a
multi-line PEM block, and hand-typing or single-line-pasting it breaks the PEM parser. Generate
and append it like this so the line breaks survive:

```bash
openssl genrsa -out /tmp/saleor-rsa.pem 2048
echo "RSA_PRIVATE_KEY=\"$(cat /tmp/saleor-rsa.pem)\"" >> infra/.env.demo
rm /tmp/saleor-rsa.pem
```

Don't try to wire `RSA_PRIVATE_KEY` through `${RSA_PRIVATE_KEY}`-style interpolation in the
compose file — Compose does that substitution as raw text *before* parsing YAML, so a
multi-line value corrupts the file. This is why `api`/`worker` in the compose file load
`infra/.env.demo` via `env_file:` instead — that path passes the value straight to the
container without going through YAML.

## 3. Bring up everything except the storefront, in this order

The storefront's build step runs GraphQL codegen against the **live**, **public**, **HTTPS**
API URL (`https://api.chidambarams.in/graphql/`) baked into `NEXT_PUBLIC_SALEOR_API_URL`.
That means the API must already be reachable over real HTTPS with a valid cert before the
storefront can build — build it too early and codegen fails with a schema-load error. So:

```bash
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo up -d db cache api worker dashboard mailpit
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo run --rm api python manage.py migrate
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo run --rm api python manage.py populatedb --createsuperuser
```

Change the seeded `admin@example.com` / `admin` credentials immediately — see
"Post-setup: securing the admin account" below.

## 4. Nginx + Certbot for all five hostnames

```bash
sudo apt install -y certbot python3-certbot-nginx apache2-utils
```

Create `/etc/nginx/sites-available/<domain>.conf` with one `server` block per subdomain
(`api`, `shop`, `admin`, `mail`), each proxying to its container's host port
(`127.0.0.1:8001`, `:3000`, `:9000`, `:8025` respectively — see the actual `ports:` mappings
in `infra/docker-compose.demo.yml` if you changed any due to a port conflict), plus a plain
redirect block for the bare domain → `shop.<domain>`. `mail.` should carry
`auth_basic`/`auth_basic_user_file` — Mailpit must never be open to the public internet.

The **API block also needs a `/media/` location** (see step 6 — do that before running
Certbot so it's covered by the same pass, or add it and reload afterward).

```bash
sudo htpasswd -c /etc/nginx/.htpasswd-mail <username>   # prompts for a password
sudo ln -s /etc/nginx/sites-available/<domain>.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d api.<domain> -d shop.<domain> -d admin.<domain> -d mail.<domain>
sudo certbot --nginx -d <domain> -d www.<domain>
```

Verify before moving on — this is the check that actually catches a misconfigured vhost:

```bash
curl -sI https://api.<domain>/graphql/
```

Expect `405` (GraphQL only accepts `POST`) with **no certificate error**. A cert mismatch here
means a server block is missing or Certbot targeted the wrong domain.

## 5. Build and start the storefront

Only once step 4's curl check is clean:

```bash
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo build storefront
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo up -d storefront
```

## 6. Media files: bind mount, not a named Docker volume

`infra/docker-compose.demo.yml` mounts `api`/`worker`'s `/app/media` to a plain host folder
(`infra/media`), not a Docker-managed named volume. This is deliberate: Django only serves
`/media/` itself when `DEBUG=True` (checked in `saleor/urls.py`), and this stack runs with
`DEBUG=False`, so Nginx has to serve media directly. A named volume's actual files live under
`/var/lib/docker/volumes/...`, which Nginx (running as an unprivileged user) generally can't
read without loosening permissions on Docker's shared data directory — too broad a blast
radius if this VM hosts other containers too. A project-local bind mount keeps the permission
fix scoped to just this folder.

```bash
mkdir -p infra/media
sudo chmod -R a+rwX infra/media
```

Add to the `api.<domain>` Nginx block, **above** the existing `location /`:

```nginx
location /media/ {
    alias /absolute/path/to/chidambarams/apps/chidambarams/infra/media/;
}
```

**If this repo checkout lives under a restricted home directory** (e.g. `/root`, which is
typically mode `700`), Nginx can't traverse into it no matter what `infra/media` itself is set
to — the block happens higher up the path. Either deploy outside of a locked-down home
directory, or grant execute-only (traversal, not listing) permission on each ancestor
directory:

```bash
sudo chmod o+x /root /root/chidambarams /root/chidambarams/apps /root/chidambarams/apps/chidambarams /root/chidambarams/apps/chidambarams/infra
```

Reload and verify:

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -sI https://api.<domain>/media/thumbnails/products/<any-seeded-image>.webp
```

Expect `200`. A `403` here means a permission problem somewhere in the path (use
`namei -l <full path to the file>` to see exactly which directory in the chain is blocking
it) — not a Nginx config problem.

## Post-setup: securing the admin account

`populatedb --createsuperuser` seeds `admin@example.com` / `admin`. Change both immediately —
this is a publicly reachable instance the moment `admin.<domain>` resolves:

```bash
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo run --rm api python manage.py shell -c "
from django.contrib.auth import get_user_model
User = get_user_model()
u = User.objects.get(email='admin@example.com')
u.email = 'your-real-email@example.com'
u.set_password('a-strong-password')
u.save()
print('Updated:', u.email)
"
```

Doing this via the Django shell (rather than the Dashboard UI's own password-reset/email-change
flows) avoids depending on outbound email delivery working correctly on a fresh box, and
updates both fields atomically in one step.

## Verification checklist

- [ ] `https://<domain>/` redirects to `https://shop.<domain>/`
- [ ] `https://shop.<domain>/` loads the storefront with product images (not broken image icons)
- [ ] `https://admin.<domain>/` — dashboard login works with the **changed** credentials
- [ ] `https://api.<domain>/graphql/` returns `405` for a bare `GET`/`HEAD`, no cert error
- [ ] `https://mail.<domain>/` prompts for basic auth before showing Mailpit

## Publishing changes

Phase 1 doesn't call for CI/CD yet — deploy manually:

```bash
cd ~/chidambarams/apps/chidambarams
git pull
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo up -d --build
```

If a service's env or volume config changed (not just image/code), Compose usually recreates it
automatically — but a service stuck in a crash-restart loop can sometimes win a race against
`up -d`'s recreate. If that happens:

```bash
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo stop <service>
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo rm -f <service>
docker compose -f infra/docker-compose.demo.yml --env-file infra/.env.demo up -d <service>
```
