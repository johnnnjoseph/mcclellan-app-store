# McClellan Labs — Umbrel Community App Store

A personal [Umbrel](https://umbrel.com) community app store. Apps here are
packaged by me, for me. Nothing is vetted by the Umbrel team.

**Store ID:** `mcclellan` — every app folder must be named `mcclellan-<app>`.

## Apps

| App | Description |
|---|---|
| [openGym](https://github.com/DuarteSantos8/openGym) | Self-hosted gym & body-weight tracker with passkey login |

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
`ts.net` name — without exposing anything to the internet.

**In the Tailscale admin console**, enable both:
- **MagicDNS** (DNS tab)
- **HTTPS Certificates** (DNS tab)

**On the Umbrel**, SSH in (`ssh umbrel@umbrel.local`) and run:

```bash
sudo tailscale serve --bg --https=443 http://localhost:8095
```

`8095` is the `port:` from `mcclellan-opengym/umbrel-app.yml`. If you changed
it, change it here too.

Check it with:

```bash
sudo tailscale serve status
```

Now open `https://umbrel.tail1a2b3c.ts.net` from any device on your tailnet and
register your passkey. Because `RP_ID` matches the hostname and the connection
is real HTTPS, it will work.

> Serving on `--https=443` maps the *whole* hostname to openGym. If you later
> want several apps on HTTPS, give each one a path instead:
> `sudo tailscale serve --bg --https=443 --set-path=/gym http://localhost:8095`
> — but note that openGym expects to live at the root, so keep it on `443`
> unless you know the app supports a subpath.

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

- Images are pinned to `:latest` because upstream doesn't publish versioned
  tags. Bumping `version:` in the manifest is what forces a re-pull.
- Umbrel's own password prompt is disabled (`PROXY_AUTH_ADD: "false"`) since
  openGym has passkey auth and invite-only signup. Flip it to `"true"` for a
  second layer.
- No app icon is set, so the tile renders blank. To fix, add
  `icon: https://...` (a direct link to an SVG or PNG) to `umbrel-app.yml`.
- Data lives in three places under the app's data directory: `data/app` (your
  profiles, passkeys and workout history — **this is the one to back up**),
  plus `data/media/img` and `data/media/gif`, which are just the downloaded
  exercise dataset and can be regenerated.
