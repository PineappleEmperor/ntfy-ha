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
5. Open the web UI with the **Open Web UI** button (it points at
   `http://<HA-LAN-IP>:8199/`) or via your public URL.

## Ports

| Container | Host (default) | Purpose |
|-----------|----------------|---------|
| `80/tcp`  | `8199`         | ntfy HTTP API + web UI |

ntfy listens on container port **80**, published on the host as **8199**. Use
that port for everything: the **Open Web UI** button (`http://<HA-LAN-IP>:8199/`),
a browser on your LAN, and your reverse proxy / Cloudflare Tunnel.

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
| `behind_proxy` | bool | `true` | Trust `X-Forwarded-*` headers. Keep `true` when running behind a reverse proxy / Cloudflare Tunnel so rate-limiting and client IPs are correct. |
| `cache_duration` | string | `12h` | How long messages are kept in the cache (e.g. `12h`, `24h`, `48h`). |
| `admin_user` | string | *(unset)* | Optional. If set together with `admin_password` (and `auth_enabled: true`), an **admin** user is created on start if it does not already exist. |
| `admin_password` | password | *(unset)* | Password for `admin_user`. Only used the first time the user is created; changing it here later does **not** update an existing user (use `ntfy user change-pass` — see below). |
| `users` | list | `[]` | Optional. Declaratively provision scoped (non-admin) users with per-user topic ACLs (see below). No shell needed. |
| `log_level` | list | `info` | ntfy log level: `trace`, `debug`, `info`, `warn`, `error`. |

### Scoped users (`users`)

Instead of using the admin account everywhere, give each device its own login
scoped to only the topics it needs. Add entries under `users` in the add-on
**Configuration** — no shell / `docker exec`:

```yaml
users:
  - name: phone
    password: choose-a-strong-one
    topics: "hab-*"        # optional ACL topic pattern; omit for none
    access: read-only      # optional: read-only (default) | read-write | write-only | deny-all
```

Each field:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Username to create (regular, non-admin role). |
| `password` | yes | Used **only** when the user is first created. Change it later with `ntfy user change-pass <name>`; editing it here does not update an existing user. |
| `topics` | no | ACL topic pattern to grant (e.g. `hab-*`, `alerts_*`). Omit to create a login with no topic grants. |
| `access` | no | Access level for `topics`: `read-only` (default), `read-write`, `write-only`, `deny-all`. |

Users are created once; the **ACL is re-applied on every start**, so changing
`topics`/`access` here and restarting keeps the grant in sync. A phone that only
receives notifications needs just `read-only` — publishing is done by your
server/token, not the phone.

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

## No ingress / sidebar panel — why

Home Assistant **ingress** serves add-ons under a per-session sub-path such as
`/api/hassio_ingress/<token>/`. **ntfy does not support being served under a URL
sub-path** — its web app requests static assets and its API/websocket at
absolute root paths (`/static/...`, `/config.js`, `/v1/...`), which under an
ingress sub-path resolve to the HA root and 404. The panel therefore never
loaded correctly, so the add-on does **not** use ingress.

Instead, the **Open Web UI** button opens the web UI on the direct host port
(`http://<HA-LAN-IP>:8199/`). ntfy's `base-url` is a single global value that
**must** equal the public URL your clients actually use (so push, attachments
and topic links work), so it points at `base_url` (your Cloudflare hostname).

**To reach the web UI:** the **Open Web UI** button / `http://<HA-LAN-IP>:8199`
on your LAN, or your **public URL** (`base_url`, e.g.
`https://ntfy.juicebox.casa`). Point the mobile/desktop apps at the public URL.

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
Worker, Home Assistant automations, etc.).

**From the web UI (recommended).** With `auth_enabled: true` the add-on turns on
ntfy's account page (`enable-login`), so you can do this in the browser:

1. Open the web UI (**Open Web UI** button, or your public URL) and **Sign in**
   as `admin_user`.
2. Open the account menu (top right) → **Account** → **Access tokens** →
   **Create access token**.
3. Copy the `tk_…` value.

> No account menu / sign-in? You're on an add-on version before `1.1.0` (which
> added `enable-login`), or `auth_enabled` is `false`. Update the add-on and
> make sure auth is on. The **Users** list under **Settings** is *not* this — it
> is only a client-side store of credentials the web app uses to read topics.

**From the CLI (fallback).** Run it inside the add-on container — e.g. from a
terminal with Docker access on the HA host:

```bash
docker exec $(docker ps -qf name=ntfy) \
  ntfy token add --config /etc/ntfy/server.yml <admin_user>
# → tk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

The `--config /etc/ntfy/server.yml` is required so ntfy finds the auth file.

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

* **No ntfy entry in the HA sidebar** — expected; the add-on does not use
  ingress (see above). Use the **Open Web UI** button or `http://<HA-IP>:8199`.
* **`401`/`403` when publishing** — with `auth_default_access: deny-all` you
  must authenticate with a user/token that has write access to the topic.
* **iOS notifications not arriving instantly** — ensure `upstream_base_url` is
  set (default `https://ntfy.sh`) and `base_url` is correct and reachable.
* **Admin password changed in options but login still uses the old one** — the
  add-on only creates the user once; change it with
  `ntfy user change-pass <admin_user>`.
