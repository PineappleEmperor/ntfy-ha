# ntfy add-on

Self-hosted [ntfy](https://ntfy.sh) push notification server for Home Assistant
OS / Supervised. ntfy lets you send push notifications to your phone or desktop
with a simple HTTP `PUT`/`POST`, and subscribe to topics from the ntfy mobile
and desktop apps.

All state (the auth database, the message cache and the attachment cache) is
stored in the add-on's persistent `/data` volume, so it survives restarts and
updates.

## Installation

1. In Home Assistant go to **Settings → Add-ons → Add-on Store**.
2. Click the ⋮ menu (top right) → **Repositories**, and add:

   ```
   https://github.com/PineappleEmperor/ntfy-ha
   ```

3. The **ntfy** add-on now appears under *Pineapple HA Add-ons*. Open it and
   click **Install**.
4. Set the options (see below), then **Start** the add-on.
5. Open the web UI from the sidebar (ingress) or via your public URL.

## Ports

| Container | Host (default) | Purpose |
|-----------|----------------|---------|
| `80/tcp`  | `8199`         | ntfy HTTP API + web UI |

ntfy listens on container port **80**. That port is:

* used internally by Home Assistant **ingress** (sidebar panel), and
* published on the host as **8199** so a reverse proxy / Cloudflare Tunnel can
  reach it.

You can change the host port `8199` to anything you like in the add-on's
**Network** tab. If you do, remember to update the matching target in your
reverse proxy / Cloudflare Tunnel (below).

## Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `base_url` | string | `https://ntfy.juicebox.casa` | The canonical public URL clients use to reach this server. **Must match** the hostname you expose via Cloudflare / your reverse proxy. Drives topic links, attachment URLs and iOS/Firebase push. |
| `upstream_base_url` | string | `https://ntfy.sh` | Upstream server used to wake up iOS devices via the ntfy.sh push relay. Leave as-is unless you have your own upstream. Clear it to disable. |
| `auth_enabled` | bool | `true` | Enable the user/ACL system (persistent auth database at `/data/user.db`). If `false`, the server is fully open (read-write for everyone). |
| `auth_default_access` | list | `deny-all` | Default access for users/anonymous clients with no explicit ACL: `deny-all`, `read-only`, `read-write`, `write-only`. `deny-all` is the safe default for a public server. |
| `behind_proxy` | bool | `true` | Trust `X-Forwarded-*` headers. Keep `true` when running behind ingress and/or a reverse proxy so rate-limiting and client IPs are correct. |
| `cache_duration` | string | `12h` | How long messages are kept in the cache (e.g. `12h`, `24h`, `48h`). |
| `admin_user` | string | *(unset)* | Optional. If set together with `admin_password` (and `auth_enabled: true`), an **admin** user is created on start if it does not already exist. |
| `admin_password` | password | *(unset)* | Password for `admin_user`. Only used the first time the user is created; changing it here later does **not** update an existing user (use `ntfy user change-pass` — see below). |
| `log_level` | list | `info` | ntfy log level: `trace`, `debug`, `info`, `warn`, `error`. |

### About the admin user

On start, if `auth_enabled` is `true` and both `admin_user` and
`admin_password` are set, the add-on creates that user as an **admin**
(read-write to all topics) if it does not already exist. This is idempotent:
it will not overwrite or re-create an existing user, and it will not fail if the
user is already present.

To manage users after that, open a terminal in the add-on
(Advanced → *not available on all installs*) or use the **SSH & Web Terminal**
add-on to `docker exec` into the container, then run e.g.:

```bash
ntfy user --config /etc/ntfy/server.yml list
NTFY_PASSWORD='newpass' ntfy user --config /etc/ntfy/server.yml change-pass admin
ntfy user --config /etc/ntfy/server.yml add somephone           # a regular user
ntfy access --config /etc/ntfy/server.yml somephone 'alerts_*' rw
```

## Home Assistant ingress (sidebar panel) — important limitation

The add-on enables ingress so ntfy shows up in the HA sidebar. However, **ntfy
does not support being served under a URL sub-path**, and Home Assistant ingress
serves add-ons under a per-session path such as
`/api/hassio_ingress/<token>/`. As a result the ingress panel may fail to load
the web app's static assets or make its API/websocket calls correctly.

Additionally, ntfy's `base-url` is a single global value and **must** equal the
public URL your clients actually use (so push, attachments and topic links
work). It therefore points at `base_url` (your Cloudflare hostname), not at the
ingress URL.

**Recommendation:** treat the sidebar panel as a convenience shortcut, and use
the **public URL** (`base_url`, e.g. `https://ntfy.juicebox.casa`) — or the
direct host port `http://<HA-LAN-IP>:8199` on your LAN — as the real way to
reach the web UI and to point the mobile/desktop apps at. This is the supported,
fully-working path.

## Exposing ntfy publicly with a Cloudflare Tunnel (manual)

You reach ntfy from the internet through your existing Cloudflare Tunnel. The
add-on does **not** touch Cloudflare — do this once in the Cloudflare
dashboard:

1. Log in to **Cloudflare Zero Trust** → **Networks → Tunnels** (or
   **Access → Tunnels**) and open your existing tunnel.
2. Go to **Published application routes** → **Add published application** (older
   UI: the **Public Hostname** tab → **Add a public hostname**).
3. Configure the route:
   * **Subdomain / Domain:** `ntfy` on `juicebox.casa` → public hostname
     `ntfy.juicebox.casa` (this must match the `base_url` option).
   * **Service type:** `HTTP`
   * **URL:** `http://<HA-host-LAN-IP>:8199`
     (e.g. `http://192.168.1.10:8199` — the LAN IP of the machine running Home
     Assistant, and the host port from the add-on **Network** tab).
4. Save. Cloudflare terminates TLS at the edge, so ntfy itself serves plain HTTP
   on `8199`; `base_url` stays `https://…` because that is the URL clients see.

> If you change the host port away from `8199` in the add-on **Network** tab,
> update the tunnel's **URL** target to match.

### Tokens and per-phone users

Once the admin user exists, mint an access token (for scripts, a Cloudflare
Worker, Home Assistant automations, etc.):

```bash
ntfy token add <admin_user>
# → tk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Use that token as a `Bearer` credential:

```bash
curl -H "Authorization: Bearer tk_xxxx..." \
     -d "Backup finished" https://ntfy.juicebox.casa/backups
```

Each phone should subscribe with **its own scoped user** rather than the admin
token. Create a user, grant it access only to the topics it needs, and log in
with those credentials in the ntfy app:

```bash
ntfy user --config /etc/ntfy/server.yml add myphone
ntfy access --config /etc/ntfy/server.yml myphone 'alerts_*' rw
```

## Using ntfy from Home Assistant

Point a `notify` / RESTful command at your public URL (or `http://<HA-IP>:8199`
on the LAN) and authenticate with a token. Example REST command:

```yaml
rest_command:
  ntfy_alert:
    url: "https://ntfy.juicebox.casa/home_alerts"
    method: POST
    headers:
      Authorization: "Bearer tk_xxxx..."
      Title: "Home Assistant"
    payload: "{{ message }}"
    content_type: "text/plain"
```

## Data & backups

Everything persistent lives under the add-on's `/data` volume:

* `/data/user.db` — users, ACLs and tokens (SQLite)
* `/data/cache.db` — cached messages (SQLite)
* `/data/attachments/` — cached attachments

It is included in Home Assistant add-on backups automatically.

## Troubleshooting

* **Web UI works via `8199`/Cloudflare but not in the sidebar** — expected; see
  the ingress limitation above.
* **`401`/`403` when publishing** — with `auth_default_access: deny-all` you
  must authenticate with a user/token that has write access to the topic.
* **iOS notifications not arriving instantly** — ensure `upstream_base_url` is
  set (default `https://ntfy.sh`) and `base_url` is correct and reachable.
* **Admin password changed in options but login still uses the old one** — the
  add-on only creates the user once; change it with
  `ntfy user change-pass <admin_user>`.
