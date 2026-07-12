# Changelog

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
