# Verdaccio NAS npm mirror

Personal Verdaccio registry for a NAS. It works as a stable npmjs proxy/mirror: clients install through Verdaccio, and Verdaccio caches downloaded packages in NAS storage.

> Security note: do **not** port-forward Verdaccio directly from your router to the public Internet. Keep it LAN/VPN-only, or put it behind a HTTPS reverse proxy with proper access controls.

## Files

- `compose.yaml` — Docker Compose service using the official `verdaccio/verdaccio:6` image.
- `.env.example` — optional bind/public URL settings.
- `verdaccio/conf/config.yaml` — NAS-oriented Verdaccio config.
Runtime files are intentionally ignored by git:

- `verdaccio/storage/` — cached packages and metadata.
- `verdaccio/plugins/` — optional plugins.
- `verdaccio/conf/htpasswd` — users/password hashes.

## Start

```bash
cp .env.example .env
mkdir -p verdaccio/conf verdaccio/storage verdaccio/plugins
docker compose up -d
```

Open:

```text
http://<NAS-IP>:4873/
```

If possible, edit `.env` and bind to the NAS LAN address instead of all interfaces:

```env
VERDACCIO_BIND=192.168.1.10
```

## Create the single user

The config starts with:

```yaml
max_users: 1
```

Create the first user from a client machine:

```bash
npm adduser --registry http://<NAS-IP>:4873/
```

Then lock registration:

```yaml
# verdaccio/conf/config.yaml
auth:
  htpasswd:
    max_users: -1
```

Restart:

```bash
docker compose restart verdaccio
```

## Configure npm clients

For global use on a trusted machine:

```bash
npm config set registry http://<NAS-IP>:4873/
npm login --registry http://<NAS-IP>:4873/
```

For one project only, create/update `.npmrc` in that project:

```ini
registry=http://<NAS-IP>:4873/
//<NAS-IP>:4873/:_authToken=<token-created-by-npm-login>
```

To return to npmjs directly:

```bash
npm config set registry https://registry.npmjs.org/
```

## Current security posture

The included config is intentionally strict:

- only the official npmjs uplink is used;
- `strict_ssl: true`;
- install/read requires authentication: `access: $authenticated`;
- bcrypt htpasswd hashes;
- first-user registration limit;
- JSON logs with auth/cookie redaction;
- Docker container drops Linux capabilities and uses `no-new-privileges`.

## External access recommendation

Avoid router port forwarding to `4873`. Prefer one of these:

1. VPN/Tailscale/WireGuard to your LAN, then use `http://<NAS-IP>:4873/`.
2. HTTPS reverse proxy, for example Caddy/Nginx/Traefik/NAS proxy, with firewall rules and authentication where possible.

If using HTTPS reverse proxy, set in `.env`:

```env
VERDACCIO_PUBLIC_URL=https://npm.example.lan
```

You may also need to enable `server.trustProxy` in `verdaccio/conf/config.yaml` for your proxy address.

## Operations

View logs:

```bash
docker compose logs -f verdaccio
```

Update image:

```bash
docker compose pull
docker compose up -d
```

Back up these paths regularly:

```text
verdaccio/conf/config.yaml
verdaccio/conf/htpasswd
verdaccio/storage/
```
