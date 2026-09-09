# McClellan Labs — Umbrel Community App Store

A personal [Umbrel](https://umbrel.com) community app store. Apps here are
packaged by me, for me. Nothing is vetted by the Umbrel team.

**Store ID:** `mcclellan` — every app folder must be named `mcclellan-<app>`.

## Apps

| App | Description |
|---|---|
| [openGym](https://gitlab.com/DuarteSantos8/opengym) | Self-hosted gym & body-weight tracker with passkey login |

---

## 1. Push this repo to GitHub

The repo must be **public** — umbrelOS clones it anonymously over HTTPS.

Create an empty repo named `mcclellan-app-store` on GitHub (no README, no
`.gitignore`), then:

```bash
cd ~/Documents/GitHub/mcclellan-app-store
git remote add origin https://github.com/<your-username>/mcclellan-app-store.git
git branch -M main
git push -u origin main
```

## 2. Set the passkey hostname (do this before installing)

openGym signs you in with **passkeys**, and WebAuthn only works on a secure
origin. Two values in `mcclellan-opengym/docker-compose.yml` must match the URL
you type in the browser:

```yaml
RP_ID: "app.musclemary.co"
ORIGIN: "https://app.musclemary.co"
```

`RP_ID` is the hostname alone — no scheme, no port. `ORIGIN` is the full URL,
and would carry a port if the origin used a non-standard one. Cloudflare
terminates TLS on 443, so it doesn't here.

> **Passkeys are cryptographically bound to `RP_ID`.** Changing it invalidates
> every passkey registered under the old hostname, and you cannot authenticate
> at the new one to migrate a profile. Export your data from Settings first if
> you ever change it.

## 3. Add the store to umbrelOS

App Store → **⋯** (top right) → **Community App Stores** → paste
`https://github.com/<your-username>/mcclellan-app-store` → **Add**.

openGym will appear under McClellan Labs. Install it.

First launch downloads ~140 MB of exercise images and GIFs, so give it a couple
of minutes before the UI fills in.

## 4. Publish it on app.musclemary.co with a Cloudflare Tunnel

A tunnel makes an **outbound** connection from the Umbrel to Cloudflare and
holds it open. Cloudflare accepts requests for `app.musclemary.co` and pushes
them down that connection. No open ports, no port forwarding, home IP never
published, and Cloudflare terminates TLS with a valid certificate — which is
what makes passkeys work.

### 4a. Install the connector

Install **Cloudflare Tunnel** from the official Umbrel App Store.

In the Cloudflare dashboard: **Network → Tunnels → Create Tunnel**, name it,
and copy the connector command it shows you. Paste that into the Umbrel app's
**Connector token** field and choose **Save & Restart**.

> That command contains a secret token. Treat it like a password.

### 4b. Route the hostname

In the tunnel's **Routes** tab, add a route:

| Field | Value |
|---|---|
| Hostname | `app.musclemary.co` |
| Service | `http://umbrel.local:8095` |

`8095` is the `port:` from `mcclellan-opengym/umbrel-app.yml`. If `umbrel.local`
doesn't resolve from inside the connector container, use the Umbrel's LAN IP
instead (`http://192.168.x.x:8095`) — less tidy, but it does not depend on mDNS.

Cloudflare creates the `app.musclemary.co` DNS record for you. The apex domain
is already on Cloudflare, so nothing else needs changing.

### 4c. Rate-limit the auth endpoints

Closing signup does not hide the app: the sign-in and pairing endpoints stay
reachable from the internet. `POST /api/pair/redeem` in particular needs no
session and accepts a short one-time code, which is the one thing here worth
not leaving unmetered.

Cloudflare's free tier includes rate-limiting rules. Cap requests to
`/api/pair/*` and `/api/auth/*` — a few per minute per IP is generous.

This is deliberately *not* Cloudflare Access. Access gates requests at the
edge, which the standalone mobile app's bearer-token requests cannot satisfy
without service tokens. Rate limiting gets most of the protection without
breaking non-browser clients.

### 4d. First sign-in, then lock it down

Open `https://app.musclemary.co`, confirm the padlock, and register your
passkey. **Then** close signup — in this order, because invite codes come from
the admin dashboard and you only become an admin after a profile exists:

1. Find your id: `grep -o '"id":"[^"]*"' ~/umbrel/app-data/mcclellan-opengym/data/app/db.json`
2. In `docker-compose.yml`, set `ADMIN_UIDS` to it and `INVITE_ONLY` to `"1"`
3. Bump `version:` in `umbrel-app.yml`, push, and update the app

Setting `INVITE_ONLY=1` before a profile exists locks everyone out permanently.

### Alternative: Tailscale, with nothing exposed

If you'd rather not publish it at all, the **Tailscale** app plus Tailscale
Serve gives a valid certificate on a `ts.net` hostname reachable only from your
own tailnet. Enable MagicDNS and HTTPS Certificates in the Tailscale admin
console, then:

```bash
sudo docker exec tailscale_web_1 tailscale serve --bg --https=8443 http://localhost:8095
```

Umbrel runs Tailscale as a container, not on the host, hence `docker exec`; the
container uses `network_mode: "host"`, so `localhost:8095` is the Umbrel's own
port. Serve on 8443 rather than 443 because umbrelOS already publishes its
dashboard on the tailnet hostname — and then `ORIGIN` must carry `:8443`, since
WebAuthn compares the full origin including the port. `RP_ID` stays
hostname-only either way.

The trade-off is that every device needs Tailscale running to reach the app.

---

## Updating an app

umbrelOS re-checks this repo periodically. To push an update:

1. Change whatever you need in the app folder.
2. Bump `version:` in `umbrel-app.yml` (and write a line in `releaseNotes:`).
3. Commit and push.

An **Update** button appears on the app's tile in umbrelOS. Bumping the version
is what triggers it — editing the compose file alone will not.

## Adding another app

```bash
mkdir mcclellan-<appname>
```

Each folder needs an `umbrel-app.yml` and a `docker-compose.yml`. The rules that
matter:

- **Folder name and `id:` must be identical**, and both must start with
  `mcclellan-`.
- Every app needs an `app_proxy` service with `APP_HOST` set to
  `<app-id>_<service>_1` and `APP_PORT` set to the port that service listens on
  *inside* the container.
- Don't publish `ports:` for HTTP services — `app_proxy` handles that. Only map
  raw ports for non-HTTP protocols.
- `port:` in `umbrel-app.yml` is the **host-facing** port and must be unique
  across every app installed on the Umbrel.
- Persist state to `${APP_DATA_DIR}/data/...` with bind mounts, not named
  volumes, and commit the directories (with a `.gitkeep`) so they exist at
  install time. Anything outside that path is lost on update.
- Omit the `networks:` key — umbrelOS injects one.
- Use multi-arch images (`linux/amd64` + `linux/arm64`) so the package survives
  a hardware change.

## Notes on this openGym package

- Upstream lives on **GitLab**, not GitHub, and images come from GitLab's
  container registry (`registry.gitlab.com/duartesantos8/opengym/...`). The old
  GitHub repo and its ghcr.io images were deleted; anything still pointing there
  fails the image pull with a 403.
- Images track `:latest`. Bumping `version:` in the manifest is what forces a
  re-pull and surfaces the Update button in umbrelOS.
- Guest mode is off. Signup is currently **open** so the first profile can be
  created — see 4d for closing it afterwards.
- Individual users can export their own data as JSON from Settings, which is
  the supported way to survive a hostname change.
- Umbrel's own password prompt is disabled (`PROXY_AUTH_ADD: "false"`) since
  openGym has passkey auth and invite-only signup. Flip it to `"true"` for a
  second layer.
- No app icon is set, so the tile renders blank. To fix, add
  `icon: https://...` (a direct link to an SVG or PNG) to `umbrel-app.yml`.
- Data lives under the app's data directory: `data/app` holds profiles,
  passkeys, workout history and the session secret — **this is the one to back
  up**. `data/media/img` and `data/media/gif` are the downloaded exercise
  dataset and can be regenerated. `data/coach-auth` caches AI Coach provider
  credentials and is deliberately kept outside `data/app` so a live token never
  ends up inside a backup.
- The exercise images and GIFs are © Gym visual, used under the upstream
  dataset's terms. They're downloaded at install time, not redistributed by
  this store. Reusing them elsewhere needs your own licence.
