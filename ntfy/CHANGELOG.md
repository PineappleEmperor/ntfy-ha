# Changelog

## 1.2.0

- Add a **`users`** option: declaratively provision scoped (non-admin) users
  with per-user topic ACLs straight from the add-on config — no shell /
  `docker exec` needed. Each entry is `name` + `password` (+ optional `topics`
  pattern and `access`, default `read-only`). Users are created once (password
  used only at creation); the ACL is re-applied on every start so topic/access
  changes take effect on restart. Ideal for a phone that only needs to **read**
  its notification topics (e.g. `topics: "hab-*"`).

## 1.1.0

- Enable the ntfy **web UI account page** (`enable-login: true`) whenever
  `auth_enabled` is on. You can now sign in as the admin user in the web UI and
  create/revoke **access tokens** from Account → Access tokens — no shell /
  `docker exec` needed. Signup stays disabled (private server); the admin mints
  users and tokens.

## 1.0.2

- Drop Home Assistant **ingress** / sidebar panel. ntfy cannot be served under a
  URL sub-path (its web app uses absolute asset/API paths), so the ingress panel
  never loaded correctly. The web UI is now reached via the direct host port
  (`8199`) or your public URL.
- Point the add-on **"Open Web UI"** button at `http://[HOST]:[PORT:80]/` (the
  LAN host IP + host port 8199) so it opens the working web UI.

## 1.0.1

- Bump add-on base image to `ghcr.io/hassio-addons/base:21.0.0`.
- Drop `armv7` (base 20.x+ no longer ships armv7); supported arch is now
  `aarch64` + `amd64`. Target host is amd64 (Intel N100).

## 1.0.0

Initial release.

- ntfy server **v2.26.0**, installed from the official GitHub release binary
  (checksum-verified) on the `ghcr.io/hassio-addons/base` Alpine base.
- Multi-arch: `aarch64`, `amd64`, `armv7`.
- s6-overlay v3 services: a `init-ntfy` oneshot renders `server.yml` from the
  add-on options and provisions an optional admin user; a `ntfy` longrun runs
  the server.
- Persistent auth database, message cache and attachment cache under `/data`.
- Ingress sidebar panel plus a direct host port (`80/tcp` → host `8199`) for a
  reverse proxy / Cloudflare Tunnel.
- Options: `base_url`, `upstream_base_url`, `auth_enabled`,
  `auth_default_access`, `behind_proxy`, `cache_duration`, `admin_user`,
  `admin_password`, `log_level`.
