# Pineapple HA Add-ons

Home Assistant add-on repository.

## Add-ons

### [ntfy](./ntfy)

Self-hosted [ntfy](https://ntfy.sh) push notification server. Send push
notifications to your phone or desktop with a simple HTTP request, with
persistent auth/cache/attachment storage, an ingress sidebar panel and an
optional direct host port for a reverse proxy or Cloudflare Tunnel.

- ntfy **v2.26.0**, multi-arch (`aarch64`, `amd64`, `armv7`).
- Full documentation: [ntfy/DOCS.md](./ntfy/DOCS.md)

## Installation

In Home Assistant: **Settings → Add-ons → Add-on Store → ⋮ → Repositories**,
then add:

```
https://github.com/PineappleEmperor/ntfy-ha
```

The **ntfy** add-on then appears in the store under *Pineapple HA Add-ons*.
Install it and follow [ntfy/DOCS.md](./ntfy/DOCS.md) for configuration and the
one-time Cloudflare Tunnel setup.
