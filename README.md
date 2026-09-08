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
origin. Two values in `mcclellan-opengym/docker-compose.yml` must exactly match
the URL you type in the browser:

```yaml
RP_ID: "umbrel.YOUR-TAILNET.ts.net"
ORIGIN: "https://umbrel.YOUR-TAILNET.ts.net"
```

Find your Umbrel's MagicDNS name in the Tailscale admin console (Machines tab)
— it looks like `umbrel.tail1a2b3c.ts.net`. Replace both lines, then commit and
push.

> **Passkeys are cryptographically bound to `RP_ID`.** Changing it later
> invalidates every passkey already registered. Pick the hostname you intend to
> keep.

## 3. Add the store to umbrelOS

App Store → **⋯** (top right) → **Community App Stores** → paste
`https://github.com/<your-username>/mcclellan-app-store` → **Add**.

openGym will appear under McClellan Labs. Install it.

First launch downloads ~140 MB of exercise images and GIFs, so give it a couple
of minutes before the UI fills in.

## 4. Turn on HTTPS with Tailscale Serve

Tailscale gives your Umbrel a real, publicly-trusted TLS certificate on its
`ts.net` name — without exposing anything to the internet. This is what makes
passkeys work.

### 4a. Get on the tailnet

1. Install the **Tailscale** app from the official Umbrel App Store and sign in
   (the app shows a login link to click).
2. Install Tailscale on the devices you'll use openGym from (Mac, iPhone) and
   sign in to the same account.
3. In the [Tailscale admin console](https://login.tailscale.com/admin/machines),
   confirm the Umbrel appears. Its **machine name** plus your **tailnet name**
   form the hostname — e.g. machine `umbrel` on tailnet `taild4a659.ts.net`
   gives `umbrel.taild4a659.ts.net`. This must match `RP_ID` exactly.

### 4b. Enable HTTPS certificates

In the admin console, **DNS** tab, turn on both:

- **MagicDNS**
- **HTTPS Certificates**

Without the second one, Tailscale serves plain HTTP and passkey registration
fails with an error that looks like an app bug. This is the step people skip.

### 4c. Run Tailscale Serve

Umbrel runs Tailscale as a **Docker container**, not on the host, so there is no
`tailscale` command on the Umbrel itself. Run it inside the container instead.

SSH in (`ssh umbrel@umbrel.local`, password is your umbrelOS dashboard
password), then:

```bash
sudo docker exec tailscale_web_1 tailscale serve --bg --https=443 http://localhost:8095
```

`tailscale_web_1` is the container name (umbrelOS names containers
`<app-id>_<service>_1`, and the Tailscale app's service is `web`). The
`localhost:8095` target resolves to the Umbrel's own port 8095 because the
container runs with `network_mode: "host"`. `8095` is the `port:` from
`mcclellan-opengym/umbrel-app.yml` — change it here if you changed it there.

Verify:

```bash
sudo docker exec tailscale_web_1 tailscale serve status
```

The config lives in the app's data volume, so it survives restarts and updates.
To undo it:

```bash
sudo docker exec tailscale_web_1 tailscale serve --https=443 off
```

### 4d. First sign-in

Open `https://umbrel.taild4a659.ts.net` from a device on your tailnet. Confirm
the padlock is real, **then** register your passkey — that is the irreversible
step, not any of the ones above.

> Serving on `--https=443` maps the whole hostname to openGym. Your umbrelOS
> dashboard stays reachable over plain HTTP at `http://umbrel` on the tailnet.
> To put more apps behind HTTPS later, give each a path with `--set-path=/name`
> — but openGym expects to live at the root, so leave it on `443`.

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
- Guest mode is off and signup is invite-only. Register the first profile
  yourself, find your id in `data/app/db.json`, then set `ADMIN_UIDS` in the
  compose file to get the admin dashboard.
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
